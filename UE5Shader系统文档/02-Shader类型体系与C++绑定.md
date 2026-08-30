# Shader 类型体系与 C++ 绑定

> 系列第 **2** 篇。UE 用一层 C++ 类体系把"shader 源码里的一个函数"映射成"引擎里可查找、可绑定、可序列化的对象"。本文讲清 `FShader` 家族的四个类型、声明/实现宏、Permutation 维度机制、序列化规则与绑定方式。

---

## 1. `FShader` 基类：它到底封装了什么

一个 `FShader` 实例代表**一个已编译的 shader 变体**（某个类型 + 某组 Permutation 取值）。它的职责有五项：

```
┌────────────────────────────────────────────────────────┐
│                      FShader                            │
├────────────────────────────────────────────────────────┤
│ ① 类型信息     FShaderType*（静态注册，反射用）           │
│ ② 已编译资源   FRHIShader*（RHI 层句柄）                 │
│ ③ 参数绑定表   FShaderParameterMap（名字 → 槽位）         │
│ ④ 序列化       保存/加载已编译字节码（Cook、DDC）         │
│ ⑤ 参数设置     把 C++ 值写进 Uniform Buffer / 资源槽      │
└────────────────────────────────────────────────────────┘
```

**关键认知**：`FShader` 的 C++ 实例是**每变体一份**的，而不是每帧一份。它由 ShaderMap 持有、跨帧复用，因此：

- ✅ 成员变量可以长期保存绑定信息（`FShaderParameter` 等）
- ❌ **不能**在 `FShader` 里存每帧变化的数据（要用参数结构体传）
- ⚠️ `SetParameters` 在渲染线程调用，成员访问需线程安全（只读即可）

---

## 2. 四类 Shader 的继承结构

```
                            FShader
                               │
       ┌───────────────────────┼───────────────────────┐
       │                       │                       │
  FGlobalShader          FMaterialShader        FMeshMaterialShader
                                ▲                       ▲
                                │                       │
                          （可继续派生）            （可继续派生）
```

| 类型 | 编译期依赖 | 变体维度 | 典型用途 |
|------|-----------|---------|---------|
| `FShader` | 无（很少直接用） | Permutation | 基类 |
| `FGlobalShader` | 平台 + FeatureLevel | Permutation | 后处理、全屏 Pass、Compute、工具 shader |
| `FMaterialShader` | + 材质 | 材质 × Permutation | 后处理材质、Light Function、材质驱动的无网格渲染 |
| `FMeshMaterialShader` | + 材质 + 顶点工厂 | 材质 × VF × Permutation | 深度 Pass、BasePass、阴影 Pass、自定义网格 Pass |

### 2.1 选择决策

```
你的 shader 需要材质节点的输出吗？
   │
   ├─ 否 ──▶ FGlobalShader
   │
   └─ 是 ──▶ 需要画网格（访问顶点属性/顶点工厂）吗？
                 │
                 ├─ 否 ──▶ FMaterialShader
                 └─ 是 ──▶ FMeshMaterialShader
```

### 2.2 为什么区分这么细

**因为变体数量**。

```
假设：
  10 个 Permutation 取值
× 200 个材质
× 8 种顶点工厂

FGlobalShader        → 10 个变体
FMaterialShader      → 10 × 200 = 2,000 个变体
FMeshMaterialShader  → 10 × 200 × 8 = 16,000 个变体
```

UE 通过"依赖关系"来精确控制：一个纯后处理 shader 不该为每个材质编译一份，所以它是 `FGlobalShader`。

---

## 3. 声明与实现宏

### 3.1 三对宏

| Shader 类型 | 声明宏 | 实现宏 |
|------------|--------|--------|
| `FGlobalShader` | `DECLARE_GLOBAL_SHADER(Class)` | `IMPLEMENT_GLOBAL_SHADER(Class, File, EntryPoint, Frequency)` |
| `FMaterialShader` | `DECLARE_SHADER_TYPE(Class, Material)` | `IMPLEMENT_MATERIAL_SHADER_TYPE(, Class, File, EntryPoint, Frequency)` |
| `FMeshMaterialShader` | `DECLARE_SHADER_TYPE(Class, MeshMaterial)` | `IMPLEMENT_MATERIAL_SHADER_TYPE(, Class, File, EntryPoint, Frequency)` |
| 通用 | `DECLARE_SHADER_TYPE(Class, Global)` | `IMPLEMENT_SHADER_TYPE(, Class, File, EntryPoint, Frequency)` |

> 带 `SHADER_USE_PARAMETER_STRUCT` 的现代写法，声明宏是一样的。

### 3.2 最小可编译示例

```cpp
// MyShader.h
#pragma once
#include "GlobalShader.h"
#include "ShaderParameterStruct.h"

class FMyGlobalShader : public FGlobalShader
{
public:
    DECLARE_GLOBAL_SHADER(FMyGlobalShader);
    SHADER_USE_PARAMETER_STRUCT(FMyGlobalShader, FGlobalShader);

    // 参数结构体（详见 03 篇）
    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
        SHADER_PARAMETER(FVector4f, TintColor)
        SHADER_PARAMETER_TEXTURE(Texture2D, InputTexture)
        SHADER_PARAMETER_SAMPLER(SamplerState, InputSampler)
        RENDER_TARGET_BINDING_SLOTS()
    END_SHADER_PARAMETER_STRUCT()
};

// MyShader.cpp
#include "MyShader.h"

IMPLEMENT_GLOBAL_SHADER(FMyGlobalShader,
    "/Plugin/MyPlugin/Private/MyShader.usf",   // 虚拟路径
    "MainPS",                                   // 入口函数名
    SF_Pixel);                                  // Shader 频率
```

### 3.3 Shader 频率枚举

| 枚举 | 对应阶段 |
|------|---------|
| `SF_Vertex` | 顶点着色器 |
| `SF_Pixel` | 像素（片元）着色器 |
| `SF_Geometry` | 几何着色器 |
| `SF_Hull` | 曲面细分控制着色器 |
| `SF_Domain` | 曲面细分求值着色器 |
| `SF_Compute` | 计算着色器 |
| `SF_RayGen` / `SF_RayHitGroup` / `SF_RayMiss` 等 | 光追阶段（按版本可用） |

### 3.4 跨模块导出

自定义 shader 类若定义在模块 A、要用在模块 B，需要用 **Exported** 变体：

```cpp
// 头文件（模块 A，需定义 MYMODULE_API）
class FMyShader : public FGlobalShader
{
public:
    DECLARE_EXPORTED_SHADER_TYPE(FMyShader, Global, MYMODULE_API);
    ...
};

// 实现（模块 A）
IMPLEMENT_EXPORTED_SHADER_TYPE(, FMyShader,
    TEXT("/Plugin/MyPlugin/Private/My.usf"),
    TEXT("MainCS"), SF_Compute, MYMODULE_API);
```

> 省略 `EXPORTED` 时，shader 类型只在当前模块可见，`GetShader<FMyShader>()` 在其他模块可能查不到（表现为"shader 为 null"）。

### 3.5 模板化的 Shader 类

```cpp
template<bool bUseHDR>
class FMyTemplatedShader : public FGlobalShader
{
    DECLARE_SHADER_TYPE(FMyTemplatedShader, Global);
    ...
};

IMPLEMENT_SHADER_TYPE(template<>, FMyTemplatedShader<true>,
    TEXT("/Plugin/MyPlugin/Private/My.usf"), TEXT("MainPS"), SF_Pixel);
IMPLEMENT_SHADER_TYPE(template<>, FMyTemplatedShader<false>,
    TEXT("/Plugin/MyPlugin/Private/My.usf"), TEXT("MainPS"), SF_Pixel);
```

> **模板 vs Permutation**：优先用 Permutation（更省代码、便于裁剪）。只有当"模板参数会影响 C++ 成员类型"（不只是宏）时才用模板。

---

## 4. Permutation 维度系统

Permutation 是 UE 表达"**编译期分支**"的标准方式：一个维度 = 一个 HLSL 宏；一组维度取值 = 一个 shader 变体。

### 4.1 声明维度

```cpp
class FMyShader : public FGlobalShader
{
public:
    DECLARE_GLOBAL_SHADER(FMyShader);
    SHADER_USE_PARAMETER_STRUCT(FMyShader, FGlobalShader);

    // ---- 维度定义 ----
    class FUseHDRDim    : SHADER_PERMUTATION_BOOL("USE_HDR");
    class FQualityDim   : SHADER_PERMUTATION_INT("QUALITY_LEVEL", 3);   // 0,1,2
    class FBlendModeDim : SHADER_PERMUTATION_ENUM_CLASS("BLEND_MODE", EMyBlendMode);

    // ---- 维度域（组合）----
    using FPermutationDomain = TShaderPermutationDomain<
        FUseHDRDim, FQualityDim, FBlendModeDim>;

    // ---- 裁剪 ----
    static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters& Parameters)
    { ... }
};
```

| 宏 | 生成的 HLSL | 取值数 |
|----|------------|-------|
| `SHADER_PERMUTATION_BOOL(Name)` | `#define Name 0/1` | 2 |
| `SHADER_PERMUTATION_INT(Name, Count)` | `#define Name 0..Count-1` | Count |
| `SHADER_PERMUTATION_ENUM_CLASS(Name, EnumType)` | `#define Name <枚举值>` | `EnumType` 的项数 |

> 另有变体（如 `SHADER_PERMUTATION_RANGE_INT`、`SHADER_PERMUTATION_SPARSE_BOOL` 等）用于精细控制取值范围与编号，名称随版本演进，以引擎源码 `ShaderPermutation.h` 为准。

### 4.2 使用维度

```cpp
// ① 构造一组取值
FMyShader::FPermutationDomain PermutationVector;
PermutationVector.Set<FMyShader::FUseHDRDim>(true);
PermutationVector.Set<FMyShader::FQualityDim>(2);

// ② 取回
const bool bHDR = PermutationVector.Get<FMyShader::FUseHDRDim>();

// ③ 转成 ID 用于查找
TShaderMapRef<FMyShader> Shader(GetGlobalShaderMap(GMaxRHIFeatureLevel),
                                PermutationVector);
```

Shader 侧：

```hlsl
void MainPS(...)
{
    float3 Color = ...;
#if USE_HDR
    Color = Tonemap(Color);
#endif

#if QUALITY_LEVEL >= 2
    Color = DoExpensiveThing(Color);
#endif
    OutColor = float4(Color, 1);
}
```

> ⚠️ **用 `#if` 而不是 `if`**：Permutation 值在编译期已知，用 `#if` 让编译器直接消除分支；用 `if` 只是运行期分支，达不到变体优化的目的（也浪费了变体）。

### 4.3 裁剪变体（关键！）

```cpp
static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters& Parameters)
{
    FPermutationDomain PermutationVector(Parameters.PermutationId);

    // ① 平台能力裁剪
    if (!IsFeatureLevelSupported(Parameters.Platform, ERHIFeatureLevel::SM5))
        return false;

    // ② 质量档裁剪：移动端只留 0 档
    if (IsMobilePlatform(Parameters.Platform) &&
        PermutationVector.Get<FQualityDim>() > 0)
        return false;

    // ③ 维度间互斥
    if (PermutationVector.Get<FUseHDRDim>() &&
        PermutationVector.Get<FQualityDim>() == 0)
        return false;   // HDR 不支持最低画质

    return true;
}
```

**裁剪前后**：

| 维度组合 | 理论 | 裁剪后 |
|---------|------|-------|
| HDR(2) × Quality(3) × Blend(4) | 24 | 可能只有 8 |

> **投入产出比**：这是治理编译时间**最优先**的一步，比任何缓存都有效。

### 4.4 三种 PermutationParameters

| 类型 | 额外提供 |
|------|---------|
| `FGlobalShaderPermutationParameters` | `Platform`、`PermutationId` |
| `FMaterialShaderPermutationParameters` | + `MaterialParameters`（材质属性：是否半透明、Shading Model...） |
| `FMeshMaterialShaderPermutationParameters` | + `VertexFactoryType` |

```cpp
// FMeshMaterialShader 的典型裁剪
static bool ShouldCompilePermutation(const FMeshMaterialShaderPermutationParameters& Parameters)
{
    // 只对不透明材质编译这个 Pass
    if (IsTranslucentBlendMode(Parameters.MaterialParameters.BlendMode))
        return false;

    // 只对支持 instancing 的 VF 编译
    if (!Parameters.VertexFactoryType->SupportsInstancing())
        return false;

    return true;
}
```

---

## 5. 序列化：`LAYOUT_FIELD` 与参数结构体

### 5.1 传统方式（显式绑定 + 序列化）

```cpp
class FMyLegacyShader : public FGlobalShader
{
    DECLARE_SHADER_TYPE(FMyLegacyShader, Global);
public:
    FMyLegacyShader() {}
    FMyLegacyShader(const ShaderMetaType::CompiledShaderInitializerType& Initializer)
        : FGlobalShader(Initializer)
    {
        // 按名字从 ParameterMap 里绑定
        TintColorParameter.Bind(Initializer.ParameterMap, TEXT("TintColor"));
        InputTextureParameter.Bind(Initializer.ParameterMap, TEXT("InputTexture"));
        InputSamplerParameter.Bind(Initializer.ParameterMap, TEXT("InputSampler"));
    }

    // ★ LAYOUT_FIELD：声明参与序列化布局的成员
    LAYOUT_FIELD(FShaderParameter, TintColorParameter);
    LAYOUT_FIELD(FShaderResourceParameter, InputTextureParameter);
    LAYOUT_FIELD(FShaderResourceParameter, InputSamplerParameter);

    virtual bool Serialize(FArchive& Ar) override
    {
        bool bShaderHasOutdatedParameters = FGlobalShader::Serialize(Ar);
        Ar << TintColorParameter;
        Ar << InputTextureParameter;
        Ar << InputSamplerParameter;
        return bShaderHasOutdatedParameters;
    }
};
```

设置参数：

```cpp
void SetParameters(FRHICommandList& RHICmdList, FRHIShader* ShaderRHI,
                   const FLinearColor& Tint, FRHITexture* Tex, FRHISamplerState* Sampler)
{
    FRHIBatchedShaderParameters& BatchedParameters = RHICmdList.GetScratchShaderParameters();
    SetShaderValue(BatchedParameters, TintColorParameter, (FVector4f)Tint);
    SetTextureParameter(BatchedParameters, InputTextureParameter, Tex);
    SetSamplerParameter(BatchedParameters, InputSamplerParameter, Sampler);
    RHICmdList.SetBatchedShaderParameters(ShaderRHI, BatchedParameters);
}
```

### 5.2 现代方式（参数结构体）✅ 推荐

```cpp
class FMyShader : public FGlobalShader
{
public:
    DECLARE_GLOBAL_SHADER(FMyShader);
    SHADER_USE_PARAMETER_STRUCT(FMyShader, FGlobalShader);   // ← 一行搞定

    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
        SHADER_PARAMETER(FVector4f, TintColor)
        SHADER_PARAMETER_TEXTURE(Texture2D, InputTexture)
        SHADER_PARAMETER_SAMPLER(SamplerState, InputSampler)
        RENDER_TARGET_BINDING_SLOTS()
    END_SHADER_PARAMETER_STRUCT()
    // 无 LAYOUT_FIELD、无 Serialize、无手写 Bind
};
```

**对比**：

| 维度 | 传统 | 参数结构体 |
|------|------|-----------|
| 绑定代码量 | 每个参数 1 行 Bind + 1 行 Serialize | **0** |
| 参数忘记绑定 | 静默失败（值为 0） | 编译期元数据校验 |
| RDG 集成 | 手动 | **自动**（`SHADER_PARAMETER_RDG_*`） |
| 反射/校验 | 弱 | 强（`FShaderParametersMetadata`） |
| 学习成本 | 低 | 中 |

> **结论**：新代码一律用参数结构体。传统方式只在维护老代码时遇到。

### 5.3 `Serialize` 的返回值

```cpp
virtual bool Serialize(FArchive& Ar) override
{
    bool bShaderHasOutdatedParameters = FGlobalShader::Serialize(Ar);
    // 若你在新版本里"改了参数布局但保持向后兼容"，这里返回 true 可触发重建
    return bShaderHasOutdatedParameters;
}
```

返回 `true` 表示"这个 shader 的参数布局已过时"，引擎会重新编译而不是加载旧的。

---

## 6. 参数绑定：名字必须对上

### 6.1 绑定链路

```
HLSL:  float4 TintColor;
          │
          │ 编译 → FShaderCompilerOutput::ParameterMap
          ▼
   ParameterMap["TintColor"] = { UB 索引, 偏移, 大小 }
          │
          │ C++ 侧元数据（宏生成）按名字查表
          ▼
   写入 Uniform Buffer 的对应字节
```

> ⚠️ **最常见的坑**：C++ 参数名与 HLSL 变量名**大小写/拼写不一致** → 查表失败 → 该参数保持 0。
> 排查方法：开启 `r.DumpShaderDebugInfo=1`，查看 `ParameterMap` 转储。

### 6.2 绑定时机

| 方式 | 时机 |
|------|------|
| 传统 `FShaderParameter::Bind` | 构造 `FShader` 时（`CompiledShaderInitializerType`） |
| 参数结构体 | 由框架在 RDG Pass 执行前自动完成 |

### 6.3 Uniform Buffer 参数的两种形式

```cpp
// ① 值类型（内联进父 Uniform Buffer）
SHADER_PARAMETER_STRUCT(FMyNestedStruct, Nested)

// ② 引用类型（独立 Uniform Buffer，靠名字绑定）
SHADER_PARAMETER_STRUCT_REF(FViewUniformShaderParameters, View)
```

`View` 是 UE 内置的全局 Uniform Buffer，几乎所有 shader 都要用：

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
    SHADER_PARAMETER_STRUCT_REF(FViewUniformShaderParameters, View)
    ...
END_SHADER_PARAMETER_STRUCT()
```

对应 HLSL 里直接可用：

```hlsl
#include "/Engine/Private/Common.ush"

float4x4 M = View.ViewToClip;
float4 Size = View.ViewSizeAndInvSize;
```

---

## 7. ShaderMap 查找

### 7.1 `TShaderMapRef`

```cpp
// ① 全局 shader（默认 permutation）
TShaderMapRef<FMyGlobalShader> Shader(GetGlobalShaderMap(GMaxRHIFeatureLevel));

// ② 全局 shader（指定 permutation）
FMyGlobalShader::FPermutationDomain Perm;
Perm.Set<FMyGlobalShader::FQualityDim>(2);
TShaderMapRef<FMyGlobalShader> Shader2(GetGlobalShaderMap(GMaxRHIFeatureLevel), Perm);

// ③ 使用
FRHIShader* ShaderRHI = Shader.GetShader();          // 拿 RHI 句柄
const FShader& Ref    = Shader.GetShaderRef();        // 拿引用
```

> `TShaderMapRef` 会缓存查找结果，避免每帧重复查找。**不要在热路径里反复构造**——把它作为 Pass 的局部变量即可（查找本身很便宜）。

### 7.2 材质 shader 查找

```cpp
const FMaterial& Material = MaterialRenderProxy->GetMaterialWithFallback(FeatureLevel, MaterialRenderProxy);

// 材质 shader（无 VF）
const TShaderRef<FMyMaterialShader> MS =
    Material.GetShaderMap()->GetShader<FMyMaterialShader>();

// Mesh 材质 shader（带 VF）
FMyMeshVS::FPermutationDomain Perm;
Perm.Set<FMyMeshVS::FSomeDim>(true);

const TShaderRef<FMyMeshVS> VS =
    Material.GetShaderMap()->GetShader<FMyMeshVS>(VertexFactoryType, Perm);
```

> ⚠️ **返回可能为空**：若 `ShouldCompilePermutation` 裁剪掉了这个组合，或材质还没编译完，`GetShader` 返回空引用。使用前务必检查：
> ```cpp
> if (VS.IsNull()) { /* 跳过这个 draw */ }
> ```

### 7.3 编译完成的判断

```cpp
// 材质是否编译完成（异步编译场景下很重要）
if (!Material.IsComplete()) return;   // 或者用默认的 fallback 材质
```

---

## 8. 生命周期与线程

### 8.1 谁持有 `FShader`

```
FShaderMapContent（ShaderMap）
        │  持有 TUniquePtr<FShader> × N
        ▼
FShaderMapBase
        │
        ▼
 全局：GetGlobalShaderMap() 的单例
 材质：FMaterialResource 持有
```

- Shader 实例的生命周期与 ShaderMap 相同
- ShaderMap 在编译完成/加载后替换
- **你拿到的 `TShaderRef` 只是引用**，不要试图手动删除

### 8.2 线程规则

| 操作 | 线程 |
|------|------|
| 编译（请求/结果写回） | 后台（SCW 进程 + 游戏线程轮询） |
| `ShouldCompilePermutation` | 任意（编译期） |
| `TShaderMapRef` 查找 | 渲染线程 |
| 参数设置 / RDG Pass 录制 | 渲染线程 |
| `FShader` 成员读取 | 渲染线程（只读） |

> ⚠️ **游戏线程改 shader 源码 / 触发重编** 时，正在被渲染线程使用的 ShaderMap 不能被立即销毁——引擎内部有延迟回收机制，你不需要（也不应该）手动处理。

---

## 9. 常见坑清单

| # | 现象 | 根因 | 对策 |
|---|------|------|------|
| 1 | shader 里参数永远是 0 | C++ 与 HLSL 名字不一致 | 逐字核对；开 `r.DumpShaderDebugInfo` |
| 2 | `GetShader<T>()` 返回空 | `ShouldCompilePermutation` 裁剪了 | 检查裁剪条件；检查 FeatureLevel |
| 3 | 编译报 "Couldn't find shader file" | 插件未注册虚拟路径 | `AddShaderSourceDirectoryMapping` |
| 4 | 模板类在其他模块查不到 | 未用 `EXPORTED` 宏 | 改用 `DECLARE_EXPORTED_SHADER_TYPE` |
| 5 | 改了 shader 不生效 | DDC 命中旧 key | `recompileshaders changed` |
| 6 | 变体数量爆炸 | Permutation 维度过多未裁剪 | `ShouldCompilePermutation` |
| 7 | 参数结构体编译不过 | 用了 `FVector`/`FMatrix`（double） | 改用 `FVector3f`/`FMatrix44f` |
| 8 | `SF_Pixel` 写成了 `SF_Vertex` | 入口函数与频率不匹配 | 核对入口函数类型 |
| 9 | 运行时卡顿 | 调用 `FinishAllCompilation()` | 改为预编译 / PSO 预热 |
| 10 | 材质 shader 拿不到参数 | 忘记 `SHADER_PARAMETER_STRUCT_REF(View...)` | 补上 View 引用 |
| 11 | Serialize 后参数错乱 | `LAYOUT_FIELD` 与 `Ar <<` 不一致 | 逐个对齐；或改用参数结构体 |
| 12 | 主机平台编译失败 | 用了非跨平台 HLSL 写法 | 用 `TEXTURE2D` / `Texture2DSample` 宏 |

---

## 10. 小结

| 主题 | 要点 |
|------|------|
| 四类 Shader | Global < Material < MeshMaterial，按"依赖什么"选择，直接决定变体数量 |
| 声明/实现宏 | `DECLARE_GLOBAL_SHADER` / `DECLARE_SHADER_TYPE` + `IMPLEMENT_*`；跨模块用 `EXPORTED` 变体 |
| Permutation | `SHADER_PERMUTATION_*` 声明维度 → HLSL 宏；`ShouldCompilePermutation` 裁剪 |
| 序列化 | 现代用参数结构体（零序列化代码）；老代码 `LAYOUT_FIELD` + `Ar <<` |
| 绑定 | 靠**参数名**查 ParameterMap；名字必须完全一致 |
| 查找 | 全局 `TShaderMapRef`；材质 `Material.GetShaderMap()->GetShader<T>()`，注意空引用 |
| 线程 | 实例跨帧复用、只读；设置参数在渲染线程 |

---

*上一篇：[01-Shader 工作原理与编译流程](01-Shader工作原理与编译流程.md) | 下一篇：[03-Shader 参数体系](03-Shader参数体系.md)*
