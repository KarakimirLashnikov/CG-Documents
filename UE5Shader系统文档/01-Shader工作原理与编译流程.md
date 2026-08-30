# Shader 工作原理与编译流程

> 系列第 **1** 篇。UE 的 shader 系统之所以"黑盒"，是因为从你连的一个材质节点到 GPU 上的一条指令，中间横跨了**代码生成、预处理、跨进程调度、多平台编译、缓存与查找**五个环节。本文逐段拆开。

---

## 1. 全链路总览

```
┌─────────────────────────────────────────────────────────────────────┐
│ ① 源码层                                                              │
│   引擎 .usf/.ush（/Engine/Private/...）                               │
│   插件 .usf/.ush（/Plugin/MyPlugin/...）                              │
│   材质编辑器生成的 HLSL 片段（运行时拼接进 MaterialTemplate.ush）       │
└───────────────────────────────┬─────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│ ② 编译输入组装（FShaderCompilerInput）                                 │
│   SourceFilename / EntryPointName / Target(平台+Shader频率)             │
│   + FShaderCompilerEnvironment {                                      │
│       Definitions: 平台宏 + FeatureLevel 宏 + 材质属性宏               │
│                    + Permutation 宏 + 自定义 SetDefine                 │
│       IncludeVirtualPathToContentsMap                                 │
│     }                                                                 │
└───────────────────────────────┬─────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│ ③ 预处理与依赖展开                                                     │
│   #include 沿虚拟路径递归展开 → 生成单个 HLSL 文件                      │
│   （用于哈希、调试转储、以及真正送进编译器）                              │
└───────────────────────────────┬─────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│ ④ 编译调度（FShaderCompilingManager → ShaderCompileWorker）            │
│   主线程不阻塞；任务按优先级入队；N 个 SCW 子进程并行编译                 │
└───────────────────────────────┬─────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│ ⑤ 平台编译                                                             │
│   Windows SM5 → FXC → DXBC                                            │
│   Windows SM6 → DXC → DXIL                                            │
│   Vulkan      → DXC(-spirv) → SPIR-V                                  │
│   Metal       → ShaderConductor(DXC → SPIR-V → SPIRV-Cross) → MSL     │
│   OpenGL(ES)  → ShaderConductor / glslang → GLSL                      │
└───────────────────────────────┬─────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│ ⑥ 编译输出（FShaderCompilerOutput）                                    │
│   Code（字节码） · ParameterMap（参数名 → 绑定槽位）                     │
│   · NumInstructions · bSucceeded · Errors                             │
└───────────────────────────────┬─────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│ ⑦ 落库与缓存                                                           │
│   写回 FShaderMap（内存） + 序列化进 DDC（磁盘/共享缓存）                 │
│   Cook 时随包烘焙为 .ushaderbytecode / .ushadercode                    │
└───────────────────────────────┬─────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│ ⑧ 运行时查找与绑定                                                     │
│   GetShader<FMyVS>(VFType, PermutationId) → TShaderRef                 │
│   → 填充参数结构体 → RDG Pass → RHI → PSO → GPU                        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. 源码层：USF / USH 与虚拟路径

### 2.1 两种文件

| 扩展名 | 含义 | 典型内容 |
|--------|------|---------|
| `.usf` | Unreal Shader **File** | 含入口函数（`MainVS` / `MainPS` / `MainCS`），可被独立编译 |
| `.ush` | Unreal Shader **Header** | 公共结构体、函数、宏，只被 `#include` |

### 2.2 虚拟路径系统

`#include` 使用的是**虚拟路径**，与磁盘路径解耦：

```
磁盘                                  虚拟路径
─────────────────────────────────    ────────────────────────────
Engine/Shaders/Private/Common.ush  →  /Engine/Private/Common.ush
Engine/Shaders/Public/Platform.ush →  /Engine/Public/Platform.ush
MyPlugin/Shaders/Private/My.usf    →  /Plugin/MyPlugin/Private/My.usf
```

引擎自带的映射在启动时注册；**插件必须自己注册**：

```cpp
// MyPlugin/Source/MyPlugin/MyPlugin.cpp
#include "Interfaces/IPluginManager.h"
#include "ShaderCore.h"          // AddShaderSourceDirectoryMapping

void FMyPluginModule::StartupModule()
{
    const FString PluginShaderDir = FPaths::Combine(
        IPluginManager::Get().FindPlugin(TEXT("MyPlugin"))->GetBaseDir(),
        TEXT("Shaders"));

    // 注册后，shader 里就能写 #include "/Plugin/MyPlugin/Private/My.usf"
    AddShaderSourceDirectoryMapping(TEXT("/Plugin/MyPlugin"), PluginShaderDir);
}
```

> ⚠️ **不注册的典型症状**：`Shader compiler error: Failed to find include file "/Plugin/MyPlugin/Private/My.usf"`。
> 另外注意 `StartupModule` 的顺序——必须在任何 shader 编译请求发出前注册。

### 2.3 常用引擎头文件

| 头文件 | 提供什么 |
|--------|---------|
| `/Engine/Public/Platform.ush` | 平台能力宏、`FEATURE_LEVEL_*`、基础数据类型 |
| `/Engine/Private/Common.ush` | View / Primitive Uniform Buffer 声明、通用数学、纹理采样宏 |
| `/Engine/Private/SceneTexturesCommon.ush` | 场景纹理的采样辅助 |
| `/Engine/Private/ScreenPass.ush` | 全屏 Pass 的顶点变换与 UV 计算 |
| `/Engine/Private/DeferredShadingCommon.ush` | GBuffer 编解码 |
| `/Engine/Private/ShadingCommon.ush` | Shading Model 枚举与判定 |
| `/Engine/Private/MaterialTemplate.ush` | **材质着色器骨架**（由材质系统填充，勿手写） |

---

## 3. 材质 → HLSL 代码生成

这是 UE 独有的环节：**材质资产本身不含 HLSL，HLSL 是编译时生成的**。

### 3.1 流程

```
UMaterial（节点图 UMaterialExpression 的 DAG）
    │
    │  UMaterial::Translate()  /  FHLSLMaterialTranslator::Translate()
    ▼
┌──────────────────────────────────────────────────────────┐
│ ① 按属性（BaseColor / Roughness / Emissive / WPO ...）逐条走图  │
│    每个节点生成一段 HLSL 表达式（内联或生成临时函数）           │
├──────────────────────────────────────────────────────────┤
│ ② 生成的代码片段填入 MaterialTemplate.ush 的占位符              │
│    float3 GetMaterialBaseColor(FMaterialPixelParameters P)     │
│    { %s }   ← 节点代码被填进 %s                                 │
├──────────────────────────────────────────────────────────┤
│ ③ 同时统计出 FMaterialCompilationOutput                        │
│    用了哪些属性、是否半透明、是否 Masked、引用了哪些纹理...       │
│    → 这些数据会变成材质属性宏（MATERIAL_*）参与后续编译           │
└──────────────────────────────────────────────────────────┘
    │
    ▼
  一份完整的 .usf 源码 + 一组宏定义
```

### 3.2 `MaterialTemplate.ush` 的地位

它是**材质着色器的公共骨架**，包含：

- `FMaterialPixelParameters` / `FMaterialVertexParameters` 结构体定义
- `GetMaterialBaseColor()` / `GetMaterialRoughness()` / ... 这些"材质查询函数"的**空壳**，由 translator 填充
- `CalcMaterialParametersEx()` 等把各属性组装成 `FMaterialParameters` 的逻辑

> **为什么重要**：你在写自己的 Mesh Material Shader 时，`#include` 了材质相关的头文件后，就能直接调用 `GetMaterialBaseColor(Parameters)`，而**不需要知道材质图长什么样**。这是 UE 材质的解耦关键。

### 3.3 材质属性宏

`FMaterialCompilationOutput` 会被转成宏定义注入编译环境：

| 宏 | 含义 | 由什么决定 |
|----|------|-----------|
| `MATERIAL_TWOSIDED` | 双面材质 | 材质勾选 Two Sided |
| `MATERIAL_MASKED` | 使用 Mask | Blend Mode = Masked |
| `MATERIAL_TRANSLUCENT` | 半透明 | Blend Mode = Translucent |
| `MATERIAL_BLEND_*` | 混合模式枚举 | Blend Mode |
| `MATERIAL_SHADINGMODEL_*` | 着色模型 | Shading Model |
| `MATERIAL_USES_SCENE_COLOR_COPY` | 需要场景色拷贝 | 材质读了 SceneColor |
| `USES_WORLD_POSITION_OFFSET` | 顶点位移 | WPO 有连接 |
| `NUM_MATERIAL_TEXCOORDS` | 使用的 UV 套数 | 节点图用到的 TexCoord 索引最大值 |
| `NUM_CUSTOM_VERTEX_INTERPOLATORS` | 自定义插值器数量 | 材质里用了 Custom UV / 自定义输出 |

> 完整列表与生成位置在 `MaterialShared.cpp` 的 `GetMaterialEnvironment()` 中，不同版本会增删。**写 shader 时用这些宏做条件编译即可**，不要硬编码。

---

## 4. 编译输入：宏定义从哪来

`FShaderCompilerEnvironment` 里的 `Definitions` 是最终送进编译器的 `-D` 集合。来源有五层：

```
┌───────────────────────────────────────────────────────────┐
│ ① 平台层        PLATFORM_WINDOWS / PLATFORM_SUPPORTS_*      │
│                由 RHI 与目标平台决定                          │
├───────────────────────────────────────────────────────────┤
│ ② FeatureLevel  FEATURE_LEVEL_ES3_1 / SM5 / SM6             │
│                COMPILE_SHADERS / USE_DEVELOPMENT_SHADERS    │
├───────────────────────────────────────────────────────────┤
│ ③ Shader 类型   由 Permutation 维度展开：USE_HDR=1, QUALITY=2 │
│                由 ModifyCompilationEnvironment 追加          │
├───────────────────────────────────────────────────────────┤
│ ④ 材质层        MATERIAL_* 系列（仅材质着色器）                │
├───────────────────────────────────────────────────────────┤
│ ⑤ 顶点工厂      USE_INSTANCING / MANUAL_VERTEX_FETCH ...     │
└───────────────────────────────────────────────────────────┘
```

### 4.1 `ModifyCompilationEnvironment`

自定义宏的标准入口：

```cpp
class FMyShader : public FGlobalShader
{
public:
    DECLARE_GLOBAL_SHADER(FMyShader);
    SHADER_USE_PARAMETER_STRUCT(FMyShader, FGlobalShader);

    static void ModifyCompilationEnvironment(
        const FGlobalShaderPermutationParameters& Parameters,
        FShaderCompilerEnvironment& OutEnvironment)
    {
        FGlobalShader::ModifyCompilationEnvironment(Parameters, OutEnvironment);

        // 追加自定义宏
        OutEnvironment.SetDefine(TEXT("MY_CUSTOM_FEATURE"), 1);
        OutEnvironment.SetDefine(TEXT("THREADGROUP_SIZE"), TEXT("64"));

        // 根据平台/FeatureLevel 分支
        if (IsFeatureLevelSupported(Parameters.Platform, ERHIFeatureLevel::SM6))
            OutEnvironment.SetDefine(TEXT("USE_WAVE_OPS"), 1);

        // 也可以追加 include 目录
        OutEnvironment.IncludeVirtualPathToContentsMap.Add(...);
    }
};
```

对应 shader 侧：

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void MainCS(uint3 DispatchThreadId : SV_DispatchThreadID)
{
#if MY_CUSTOM_FEATURE
    ...
#endif

#ifdef USE_WAVE_OPS
    ...   // WaveActiveSum 等 SM6 特性
#endif
}
```

### 4.2 `ShouldCompilePermutation`：裁剪变体

不返回 `true` 的维度组合**根本不会编译**，这是控制变体数量的第一道闸：

```cpp
static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters& Parameters)
{
    FPermutationDomain PermutationVector(Parameters.PermutationId);

    // 例：移动端只编译低质量档
    if (!IsFeatureLevelSupported(Parameters.Platform, ERHIFeatureLevel::SM5) &&
        PermutationVector.Get<FQualityDim>() > 0)
        return false;

    return true;
}
```

| 参数类型 | 用于 |
|---------|------|
| `FGlobalShaderPermutationParameters` | `FGlobalShader` |
| `FMaterialShaderPermutationParameters` | `FMaterialShader`（多出 `Parameters.MaterialParameters`） |
| `FMeshMaterialShaderPermutationParameters` | `FMeshMaterialShader`（多出 `Parameters.VertexFactoryType`） |

---

## 5. 编译调度与 ShaderCompileWorker

### 5.1 为什么是"跨进程"

```
Editor / Game 进程                      ShaderCompileWorker.exe × N
┌──────────────────────┐               ┌──────────────────────┐
│ FShaderCompilingManager│  ──IPC──▶   │ 收到编译任务           │
│  · 任务队列（有优先级） │             │  · 预处理              │
│  · 轮询结果            │  ◀──IPC──   │  · 调平台编译器         │
│  · 主线程不阻塞         │             │  · 回传字节码/错误      │
└──────────────────────┘               └──────────────────────┘
```

**收益**：
- 编译崩溃（编译器 bug、OOM）不会拖垮编辑器
- 天然的并行与隔离
- 可以跨机器分布式（配合 DDC / XGE / SN-DBS / Incredibuild）

### 5.2 关键 API

```cpp
#include "ShaderCompiler.h"

// 等待所有异步编译完成（阻塞，慎用于运行期）
GShaderCompilingManager->FinishAllCompilation();

// 轮询进行中的任务数
const int32 NumPending = GShaderCompilingManager->GetNumRemainingJobs();

// 判断某个 shader map 是否编译完成
bool bComplete = ShaderMap->IsCompilationFinalized();
```

> ⚠️ **运行期调用 `FinishAllCompilation()` 会造成明显卡顿**。正确做法是：加载时预编译 / 用 `FShaderCompilingManager::IsAsyncCompilationAllowed()` 判断、或用 PSO 预热机制。

### 5.3 任务优先级

| 优先级 | 场景 |
|--------|------|
| 高 | 编辑器里正在编辑的材质（要立刻看到结果） |
| 中 | 场景中可见对象的材质 |
| 低 | 预加载、后台编译 |

---

## 6. 平台编译与跨平台策略

### 6.1 UE 的统一策略：**只写 HLSL**

UE 的 shader 源码用 HLSL 语法书写，再按目标平台翻译：

| 目标 | 工具链 | 产物 |
|------|--------|------|
| D3D11 / SM5 | FXC | DXBC |
| D3D12 / SM6 | DXC | DXIL |
| Vulkan | DXC（`-spirv`） | SPIR-V |
| Metal（Mac/iOS） | ShaderConductor：DXC → SPIR-V → SPIRV-Cross | MSL |
| OpenGL / GLES | ShaderConductor / glslang | GLSL |
| 主机平台 | 平台厂商工具链 | 私有格式 |

```
        HLSL 源码
            │
            ▼
      ┌───────────┐
      │    DXC    │────────▶ DXIL（D3D12 SM6）
      └─────┬─────┘
            │ -spirv
            ▼
        SPIR-V ─────────────▶ SPIR-V（Vulkan 直接吃）
            │
            │ SPIRV-Cross（ShaderConductor 内）
            ▼
      MSL / GLSL
```

> **实践含义**：你写的 HLSL 会被翻译成完全不同的语言。因此：
> - **遵守 UE 的跨平台宏**（`TEXTURE2D` / `Texture2DSample` / `RWTEXTURE2D` ...），不要直接写 `Texture2D<float4>` 与 `.Sample()`
> - 某些 SM6 高级特性（Wave Ops、Bindless 的 `ResourceDescriptorHeap`）在翻译路径上不可用或行为不同，要用 `#if` 保护
> - 细节见 [06 篇](06-常用宏速查大全.md) 的 HLSL 跨平台宏表

### 6.2 Feature Level

| Feature Level | 对应 | 典型平台 |
|---------------|------|---------|
| `ERHIFeatureLevel::ES3_1` | GLES3.1 / Metal 1.x | 移动端、Switch |
| `ERHIFeatureLevel::SM5` | D3D11 级（也跑在 D3D12 上） | 老 PC、部分主机 |
| `ERHIFeatureLevel::SM6` | D3D12 Ultimate 级 | 现代 PC、次世代主机 |

Shader 里用 `FEATURE_LEVEL_SM5` 等宏做条件编译。

---

## 7. 编译输出

`FShaderCompilerOutput` 的关键内容：

| 字段 | 说明 |
|------|------|
| `Code` | 平台字节码 |
| `ParameterMap` | **参数名 → 绑定信息**（Uniform Buffer 索引、基址偏移、资源绑定槽位） |
| `NumInstructions` | 指令数（用于性能估算与统计） |
| `bSucceeded` / `Errors` | 编译结果与错误列表 |
| `bSupportsQueryingUsedAttributes` | 是否支持查询使用的顶点属性 |
| `UsedAttributes` | 实际用到的顶点属性（用于剔除顶点流） |

> **`ParameterMap` 是反射的结果**：它记录了"shader 里叫 `SimpleColor` 的参数，在 Uniform Buffer 的第几个字节"。C++ 侧的 `FShaderParameter` 在绑定时就是靠名字去这张表里查。**名字对不上 → 绑定失败 → 参数默认值出现（通常是全 0）**，这是"shader 里读到的值永远是 0"的最常见原因。

---

## 8. DDC：派生数据缓存

### 8.1 Key 的构成

```
DDC Key = Hash(
    材质 GUID / 全局 shader 类型哈希
  + 平台 + FeatureLevel
  + Shader 类型 + 入口函数名 + 频率（VS/PS/CS）
  + 顶点工厂类型（若有）
  + Permutation ID
  + 宏定义集合（Definitions）
  + 展开后的 HLSL 源码内容
  + 引擎 shader 源码版本哈希
)
```

> ⚠️ **任何一项变化都会导致 key 变化 → 重新编译**。这也解释了：改一个被几千个材质 `#include` 的 `.ush` 文件，会触发海量重编。

### 8.2 三级缓存

```
┌──────────────────────────────────────────────┐
│ L1  内存 ShaderMap    进程内，最快              │
├──────────────────────────────────────────────┤
│ L2  本地 DDC          %LOCALAPPDATA%/DerivedDataCache │
├──────────────────────────────────────────────┤
│ L3  共享 DDC          网络共享目录 / 云（团队共享）│
└──────────────────────────────────────────────┘
```

配置（`DefaultEngine.ini`）：

```ini
[InstalledDerivedDataBackendGraph]
MinimumDaysToKeepFile=7
Root=(Type=KeyLength, Length=120, Inner=AsyncPut)

[DerivedDataBackendGraph]
Root=(Type=KeyLength, Length=120, Inner=HierarchicalShared)
HierarchicalShared=(Type=Hierarchical, Inner=Boot, Inner=Pak, Inner=EnginePak, Inner=Local, Inner=Shared)
Local=(Type=FileSystem, ReadOnly=false, Clean=false, Flush=false, PurgeTransient=true, DeleteUnused=true, UnusedFileAge=34, FoldersToClean=-1, Path=../../../Engine/DerivedDataCache)
Shared=(Type=FileSystem, ReadOnly=false, Clean=false, Flush=false, DeleteUnused=true, UnusedFileAge=10, FoldersToClean=10, Path=\\MyServer\UE_DDC, EnvPathOverride=UE-SharedDataCachePath)
```

> **团队实践**：一定要配共享 DDC。一份好的共享 DDC 能把新人首启动的 shader 编译从几十分钟降到几分钟。

### 8.3 Cook 时的 shader 烘焙

打包时，shader 会被：

1. 收集：遍历所有会被使用的材质与 shader 类型
2. 编译：全量编译目标平台的所有变体
3. 去重：相同字节码合并（Share Material Shader Code）
4. 打包：写入 `.ushaderbytecode` / `.ushadercode`（Shader Code Library）

相关项目设置：

| 设置 | 位置 | 作用 |
|------|------|------|
| **Share Material Shader Code** | Project Settings → Packaging | 相同字节码只存一份，减小包体 |
| **Shared Material Native Libraries** | 同上 | 生成平台原生的共享 shader 库 |
| **Reduce Shader Permutations** | 同上（移动端/静态光照场景） | 裁剪不可能用到的变体 |

---

## 9. ShaderMap 与变体查找

### 9.1 三个层次

| 容器 | 内容 | 获取 |
|------|------|------|
| 全局 ShaderMap | 所有 `FGlobalShader` 的变体 | `GetGlobalShaderMap(FeatureLevel)` |
| 材质 ShaderMap | 某材质下所有 `FMaterialShader` / `FMeshMaterialShader` 的变体 | `MaterialResource.GetShaderMap()` |
| Shader Code Library | 打包后的物理字节码库 | 由 RHI 层管理 |

### 9.2 查找方式

```cpp
// ① 全局 shader：直接用 TShaderMapRef
TShaderMapRef<FMyGlobalShader> Shader(GetGlobalShaderMap(ERHIFeatureLevel::SM5));

// 或者显式带 permutation
FMyGlobalShader::FPermutationDomain PermutationVector;
PermutationVector.Set<FMyGlobalShader::FUseHDRDim>(true);
TShaderMapRef<FMyGlobalShader> Shader(GetGlobalShaderMap(GMaxRHIFeatureLevel), PermutationVector);
```

```cpp
// ② 材质 shader：从材质的 shader map 里取
const FMaterialRenderProxy* MaterialProxy = ...;
const FMaterial& Material = MaterialProxy->GetMaterialWithFallback(FeatureLevel, MaterialProxy);

// 全局/材质 shader（不依赖 VF）
const TShaderRef<FMyMaterialShader> Shader =
    Material.GetShaderMap()->GetShader<FMyMaterialShader>();

// Mesh 材质 shader（依赖 VF）
const TShaderRef<FMyMeshVS> Shader =
    Material.GetShaderMap()->GetShader<FMyMeshVS>(VertexFactoryType);
```

> **版本注意**：UE 5.3 起材质着色器容器类型演进为 `FMaterialShaders`，`GetShaderMap()` 的返回类型随之变化，但 `GetShader<T>()` 的用法保持一致。若你手上的版本编译不过，检查 `FMaterial::GetShaderMap()` 的返回类型名。

### 9.3 变体数量的量级

一个典型 PBR 材质的变体数：

```
材质属性组合 × VF 类型 × FeatureLevel × 平台 × Permutation 维度 × 光照/投影变体
```

| 因子 | 典型取值 |
|------|---------|
| VF 类型 | 5 ~ 15（Static / Skinned / Instanced / Landscape / Niagara ...） |
| FeatureLevel | 1 ~ 3 |
| 平台 | 1 ~ 6 |
| 材质 Permutation（含质量/光照分支） | 数十 ~ 数百 |
| Static Switch / Static Parameter | 由材质决定，可爆炸 |

> **这就是为什么 UE 项目的 shader 编译能跑几小时**。治理手段见 [08 篇](08-调试优化与最佳实践.md)。

---

## 10. PSO 与 Shader Pipeline Cache

编译完 shader 只是第一步，GPU 还需要 **Pipeline State Object**（Vulkan/D3D12/Metal 都有）。PSO 创建本身有毫秒级开销，首次用到会卡顿。

UE 的做法：**记录运行过的 PSO，下次启动时提前编译**。

```
首次运行                        后续运行
──────────────                  ──────────────
遇到新 PSO → 同步创建（卡顿）    读取 PSO 记录文件
      │                              │
      ▼                              ▼
  记录到 stablepc.csv           后台批量预热 PSO
                                      │
                                      ▼
                                命中 → 无卡顿
```

关键控制台变量：

| 变量 | 作用 |
|------|------|
| `r.ShaderPipelineCache.Enabled` | 启用 PSO 记录与预热 |
| `r.ShaderPipelineCache.LogPSO` | 记录遇到的 PSO 到磁盘 |
| `r.ShaderPipelineCache.SaveBoundPSOLog` | 退出时保存记录 |
| `r.ShaderPipelineCache.BatchSize` | 每帧预热的批大小 |
| `r.ShaderPipelineCache.StartupMode` | 启动时的预热策略 |

产物位于 `Saved/CollectedPSOs/`（`.stablepc.csv` / `.upipelinecache`）。

---

## 11. 运行时热重载

### 11.1 控制台命令

```
recompileshaders changed      # 只重编内容发生变化的 shader（最常用）
recompileshaders all          # 全量重编（慢）
recompileshaders global       # 只重编全局 shader
recompileshaders material <Name>   # 重编指定材质
```

### 11.2 编辑器热重载流程

```
修改 .usf / .ush 并保存
      │
      ▼
引擎检测到 shader 源文件变更（虚拟路径 → 磁盘时间戳）
      │
      ▼
标记依赖这些文件的所有 shader map 失效
      │
      ▼
重新组装编译输入 → 走完整编译流程 → 替换 ShaderMap
      │
      ▼
场景中所有使用该 shader 的材质自动刷新
```

### 11.3 开发模式（强烈建议开启）

```ini
; ConsoleVariables.ini 或 Engine/Config/ConsoleVariables.ini
r.ShaderDevelopmentMode=1        ; 出错时保留调试信息、支持快速迭代
r.DumpShaderDebugInfo=1          ; 转储编译中间产物
r.DumpDebugShaderText=1          ; 转储预处理后的完整 HLSL
```

开启后，编译错误会给出**展开后的行号**，中间产物落在 `Saved/ShaderDebugInfo/`。

---

## 12. 关键类速查

| 类 / 结构 | 所在模块 | 职责 |
|-----------|---------|------|
| `FShaderType` | RenderCore | shader 类型的静态注册表与元信息 |
| `FShader` | RenderCore | 一个已编译 shader 实例的基类（序列化、绑定） |
| `FShaderMapBase` / `FShaderMapContent` | RenderCore | 变体容器 |
| `FShaderCompilerInput` | ShaderCompiler | 一次编译请求的全部输入 |
| `FShaderCompilerOutput` | ShaderCompiler | 编译结果（字节码 + 参数表） |
| `FShaderCompilerEnvironment` | ShaderCompiler | 宏定义集合与 include 虚拟内容 |
| `FShaderCompilingManager` | ShaderCompiler | 编译任务调度 |
| `FShaderCompileUtilities` | ShaderCompiler | 预处理与结果解析的工具函数 |
| `FHLSLMaterialTranslator` | Engine | 材质节点图 → HLSL |
| `FMaterialCompilationOutput` | Engine | 材质编译产出的属性统计 |
| `FMaterialResource` | Engine | 一个（FeatureLevel 下的）材质渲染资源与其 shader map |
| `FMaterialShaders` | RenderCore | UE 5.3+ 的材质着色器容器 |

---

## 13. 小结

| 环节 | 要点 |
|------|------|
| 源码 | `.usf` / `.ush` + **虚拟路径**（插件必须注册映射） |
| 材质 | 节点图 → `FHLSLMaterialTranslator` 填进 `MaterialTemplate.ush` → 生成 HLSL + 属性宏 |
| 编译输入 | 五层宏：平台 / FeatureLevel / Shader 类型 / 材质 / 顶点工厂 |
| 编译调度 | 跨进程 SCW 并行；主线程不阻塞 |
| 跨平台 | 只写 HLSL → DXC → (SPIR-V) → SPIRV-Cross → MSL/GLSL |
| 输出 | 字节码 + **ParameterMap**（名字→绑定） |
| 缓存 | 三级 DDC（内存 / 本地 / 共享）+ Cook 时烘焙 |
| 查找 | 全局用 `TShaderMapRef`，材质用 `Material.GetShaderMap()->GetShader<T>()` |
| PSO | ShaderPipelineCache 记录 + 预热，消除首帧卡顿 |

---

*上一篇：[00-总览与架构](00-总览与架构.md) | 下一篇：[02-Shader 类型体系与 C++ 绑定](02-Shader类型体系与C++绑定.md)*
