# 材质系统与 HLSL 代码生成

> 系列第 **7** 篇。UE 的材质编辑器本质是一个**可视化的 HLSL 生成器**：节点图在保存/加载时被翻译成 HLSL 片段，再填入引擎的材质骨架。本文讲清这条翻译链路、材质参数如何到达 shader，以及 Custom 节点 / 材质函数 / Static Switch 这些常用扩展点的机制与坑。

---

## 1. 材质的两个世界

```
┌─────────────────────────── 游戏线程 / 编辑器 ──────────────────────────┐
│  UMaterial              资产：节点图（UMaterialExpression 的 DAG）      │
│     ▲                                                                  │
│     │ 继承                                                              │
│  UMaterialInstance     覆写参数（标量/向量/纹理/静态开关）                │
│     ▲                                                                  │
│     │ 继承                                                              │
│  UMaterialInstanceDynamic   运行时可改参数（MID）                        │
└───────────────────────────────┬──────────────────────────────────────┘
                                │  编译（CacheShaders）
                                ▼
┌─────────────────────────── 渲染线程 ──────────────────────────────────┐
│  FMaterialResource      每个 FeatureLevel 一份，持有 ShaderMap          │
│  FMaterial              渲染线程的材质接口（编译状态、属性查询）           │
│  FMaterialRenderProxy   渲染代理：把参数值送进 shader 的载体             │
└──────────────────────────────────────────────────────────────────────┘
```

| 概念 | 所在线程 | 作用 |
|------|---------|------|
| `UMaterial` | 游戏/编辑 | 资产与节点图，可编辑 |
| `FMaterialResource` | 渲染 | 编译产物 + ShaderMap（每 FeatureLevel 一份） |
| `FMaterial` | 渲染 | 只读接口：取 BlendMode、ShadingModel、ShaderMap |
| `FMaterialRenderProxy` | 渲染 | 携带**参数值**，供 Pass 绑定 |

> **一句话理解**：`UMaterial` 是"配方"，`FMaterialResource` 是"按配方做好的菜"，`FMaterialRenderProxy` 是"端上桌的那一盘（含当次的参数）"。

---

## 2. 编译触发与流程

### 2.1 何时编译

| 触发 | 场景 |
|------|------|
| 编辑器里点 Apply / 保存材质 | `UMaterial::PostEditChangeProperty` → `CacheResourceShadersForRendering` |
| 加载资产 | 反序列化后触发 |
| Cook | 全量编译目标平台 |
| `recompileshaders material <Name>` | 手动 |

### 2.2 流程

```
① UMaterial::CacheResourceShadersForRendering()
      │
      ▼
② FHLSLMaterialTranslator 遍历节点图
      │  · 对每个材质属性（BaseColor / Roughness / ...）从根节点倒着走
      │  · 每个节点 → 一段 HLSL 表达式或临时函数
      │  · 复用：相同子图会生成临时变量避免重复计算
      ▼
③ 生成的 HLSL 片段填入 MaterialTemplate.ush 的占位符
      │  float3 GetMaterialBaseColor(FPixelMaterialInputs In) { %s }
      ▼
④ 统计 FMaterialCompilationOutput
      │  · 用了哪些属性
      │  · 是否半透明 / Masked / 双面
      │  · 引用了哪些纹理（采样器数量）
      │  · 是否需要场景色 / 顶点插值器数量
      ▼
⑤ 生成宏定义集合（MATERIAL_*）→ 走 [01 篇](01-Shader工作原理与编译流程.md) 的编译流程
```

---

## 3. `MaterialTemplate.ush`：材质的骨架

它是所有材质着色器的公共模板，位置：`Engine/Shaders/Private/MaterialTemplate.ush`。

它提供三类东西：

### 3.1 参数结构体

```hlsl
struct FMaterialVertexParameters
{
    float3 WorldPosition;
    float3 WorldNormal;
    float3 VertexColor;
    float2 TexCoords[TEXCOORD_COUNT];
    ...
};

struct FMaterialPixelParameters
{
    float2 TexCoords[TEXCOORD_COUNT];
    float3 WorldPosition;
    float3 WorldNormal;
    float3 CameraVector;
    float3 VertexColor;
    float4 SvPosition;
    ...
};

struct FPixelMaterialInputs
{
    MaterialFloat3 BaseColor;
    MaterialFloat  Metallic;
    MaterialFloat  Roughness;
    MaterialFloat3 EmissiveColor;
    MaterialFloat  Opacity;
    MaterialFloat  OpacityMask;
    ...
};
```

### 3.2 查询函数（由 translator 填充实现）

```hlsl
MaterialFloat3 GetMaterialBaseColor(FPixelMaterialInputs In);
MaterialFloat  GetMaterialMetallic(FPixelMaterialInputs In);
MaterialFloat  GetMaterialRoughness(FPixelMaterialInputs In);
MaterialFloat3 GetMaterialEmissive(FPixelMaterialInputs In);
MaterialFloat  GetMaterialOpacity(FPixelMaterialInputs In);
MaterialFloat  GetMaterialOpacityMask(FPixelMaterialInputs In);
MaterialFloat3 GetMaterialNormal(FMaterialPixelParameters In, FPixelMaterialInputs PixelInputs);
MaterialFloat3 GetMaterialWorldPositionOffset(FMaterialVertexParameters In);
```

> **这些函数就是"材质图"的入口**。自定义 shader 不需要知道材质图长什么样，只要调 `GetMaterialBaseColor()`。

### 3.3 阶段分流

```hlsl
#if VERTEXSHADER
    // 顶点着色器版本的材质代码（WPO 等）
#endif

#if PIXELSHADER
    // 像素着色器版本的材质代码
#endif
```

| 宏 | 含义 |
|----|------|
| `VERTEXSHADER` | 当前编译的是 VS |
| `PIXELSHADER` | 当前编译的是 PS |
| `COMPUTESHADER` | 当前编译的是 CS（材质一般不用） |

---

## 4. 材质参数如何到 shader

### 4.1 标量 / 向量 / 纹理参数

| 材质节点 | C++ 侧设置 | shader 侧存储 |
|---------|-----------|--------------|
| ScalarParameter | `SetScalarParameterValue(Name, Value)` | 打包进 `Material` UB 的数组 |
| VectorParameter | `SetVectorParameterValue(Name, FLinearColor)` | 同上 |
| TextureParameter | `SetTextureParameterValue(Name, UTexture*)` | 绑定到材质纹理槽位 |
| StaticSwitchParameter | `SetScalarParameterValue`（需配合静态参数） | **编译期宏**（产生变体） |
| RuntimeVirtualTexture | — | 专用槽位 |

> 标量/向量参数被 translator 打包成索引数组（概念上类似 `Material.ScalarExpressions[N]` / `Material.VectorExpressions[N]`），translator 生成对应的索引访问代码。**你一般不需要直接访问这些数组**——要么在材质图里用参数节点，要么在自定义 shader 里调 `GetMaterial*()`。

### 4.2 运行时改参数（MID）

```cpp
// 创建动态材质实例
UMaterialInstanceDynamic* MID = MeshComp->CreateAndSetMaterialInstanceDynamic(0);

// 改参数
MID->SetScalarParameterValue(TEXT("Intensity"), 2.0f);
MID->SetVectorParameterValue(TEXT("TintColor"), FLinearColor::Red);
MID->SetTextureParameterValue(TEXT("Mask"), SomeTexture);
```

| 层级 | 改参数的代价 |
|------|-------------|
| `UMaterial`（改节点图） | **重编译全部变体**（秒~分钟） |
| `UMaterialInstance`（改参数值） | 无需重编译（参数已是编译期布局的一部分） |
| `UMaterialInstanceDynamic` | 零重编译，每帧改都行 |
| **Static Parameter**（改静态开关） | **产生新变体 → 重编译** ⚠️ |

> ⚠️ **最常见的性能事故**：在运行时频繁修改 `StaticSwitchParameter` 或让蓝图动态创建大量不同静态参数组合的材质实例 → 每次都触发 shader 编译 → 卡顿。

### 4.3 材质参数集合（MPC）

跨材质共享的全局参数容器：

```cpp
// C++ 侧（游戏线程）
UMaterialParameterCollectionInstance* MPCInstance =
    GetWorld()->GetParameterCollectionInstance(MPCAsset);
MPCInstance->SetScalarParameterValue(TEXT("GlobalWetness"), 0.7f);
MPCInstance->SetVectorParameterValue(TEXT("WindDirection"), FLinearColor(...));
```

材质编辑器里拖入 **Collection Parameter** 节点即可引用；HLSL 侧对应一个全局 Uniform Buffer（概念上形如 `MaterialCollection0`，具体命名与绑定以引擎为准）。

| 用途 | 说明 |
|------|------|
| 全局天气 / 时间 / 风 | 一次设置，所有材质生效 |
| 避免每个材质重复设置 | 集中管理 |
| **代价** | MPC 参数**不会**产生变体，属于运行期 uniform，开销极低 |

> ✅ **强烈建议**：能用 MPC 就用 MPC，比在每个材质里塞参数 + 每个实例上设值高效得多。

---

## 5. Custom 节点（Custom Expression）

在材质图里直接写 HLSL。

### 5.1 基本写法

```
节点设置：
  Code:      return frac(sin(dot(%s, float2(12.9898, 78.233))) * 43758.5453);
  Output Type: CMOT_Float1
  Inputs:    [0] UV (float2)
```

translator 会生成一个函数：

```hlsl
MaterialFloat CustomExpression0(FMaterialPixelParameters Parameters)
{
    return frac(sin(dot(UV, float2(12.9898, 78.233))) * 43758.5453);
}
```

### 5.2 关键规则

| 规则 | 说明 |
|------|------|
| `%s` 占位符 | 每个输入对应一个 `%s`，**必须全部用到** |
| `return` | 输出类型与 `Output Type` 必须匹配 |
| 多行代码 | 先声明变量再 return，最后一行必须 return |
| `#include` | ❌ 不能在 Code 里写；用节点的 **Include File Paths** 设置 |
| 自定义宏 | 用 Additional Defines 设置 |
| 精度 | 用 `MaterialFloat` 而非 `float`，跟随材质精度设置 |

### 5.3 常见坑

| # | 坑 | 说明 |
|---|-----|------|
| 1 | VS 与 PS 双份代码 | Custom 节点若在 VS 路径使用，会生成**两份**代码（顶点/像素各自一份），两次编译 |
| 2 | 访问 `Parameters.TexCoords[0]` | 在 Custom 节点里要用 `Parameters.TexCoords[0]` 而非 `UV`，除非你声明了输入 |
| 3 | 循环里的纹理采样 | 动态循环内采样会导致编译器**展开或降级**，性能不可控 |
| 4 | 忘记 return | 编译报错，且报错行号指向生成的代码，不是你的 Code |
| 5 | 用了非跨平台写法 | 直接写 `Texture2D.Sample()` 会在 Metal/GL 路径失败 |
| 6 | 大量 Custom 节点 | 生成的代码无法被材质编译器优化合并，性能通常劣于内置节点 |

> **建议**：Custom 节点用于**原型验证与少量特殊逻辑**。稳定后把逻辑固化成 C++ 侧的引擎 shader（Global Shader）或提交为引擎的材质节点。

---

## 6. 材质函数（Material Function）

把可复用的节点子图封装成资产，供多个材质引用。

```
MaterialFunction: MF_TriplanarMapping
   输入：Texture, Scale, BlendSharpness
   输出：float3 Color
         │
         │ 在材质图中作为一个节点使用
         ▼
   展开（内联）进引用它的材质
```

| 特性 | 说明 |
|------|------|
| **内联展开** | 不是函数调用，是代码复制 → 无调用开销，但会增加生成代码量 |
| 可嵌套 | 函数里可以再引用函数 |
| `Expose to Library` | 暴露到材质编辑器的函数库中 |
| 版本管理 | 改函数会触发**所有引用它的材质重编译** |

> ⚠️ 一个被 200 个材质引用的核心材质函数，改一行会导致 200 个材质重编译。慎重对待高频复用函数的修改。

---

## 7. Static Switch 与变体

`StaticSwitchParameter` / `StaticBoolParameter` 会在**编译期**决定代码分支，产生**独立的 shader 变体**。

```
材质图中： StaticSwitch(UseDetailNormal) → 分支 A / 分支 B
                    │
                    ▼ 编译期
    ┌───────────────┴───────────────┐
    ▼                               ▼
  变体 1（USE_DETAIL=0）          变体 2（USE_DETAIL=1）
```

| 维度 | Static Switch | 普通 if / lerp |
|------|--------------|---------------|
| 决策时机 | 编译期 | 运行期 |
| 性能 | 无分支开销（代码被消除） | 有分支/双路计算开销 |
| 代价 | **变体数量翻倍** | 无额外变体 |
| 修改代价 | 触发重编译 | 无 |

### 7.1 变体数量估算

```
N 个 Static Switch → 最多 2^N 个变体

N=3   →     8
N=5   →    32
N=8   →   256   ← 已经开始危险
N=12  →  4096   ← 灾难
```

> 再乘以 VF 类型 × FeatureLevel × 平台 × 光照/阴影变体 —— 这就是 UE 项目 shader 编译慢的根源。

### 7.2 治理建议

| 策略 | 做法 |
|------|------|
| **减少 Static Switch** | 3 个以内；更多的用运行期分支或 uniform |
| **用 MPC 代替** | 全局开关走 MPC（uniform），不产生变体 |
| **用 Quality Switch** | 质量相关的分支用 Quality Switch（引擎已内建处理） |
| **材质实例不要乱覆写静态参数** | 每覆写一次就多一个变体 |
| **检查静态参数集** | 编辑器里能看到材质当前的 Static Parameter Set |

### 7.3 Quality Switch / Feature Level Switch

| 节点 | 用途 |
|------|------|
| **Quality Switch** | 按画质档位（Low/Medium/High）选择不同分支，引擎按档位编译 |
| **Feature Level Switch** | 按 Feature Level（ES3.1/SM5/SM6）选择分支 |
| **Platform Switch**? | 部分版本提供按平台的分支 |

> 这两个节点本质也是编译期分支，但引擎对它们做了统一管理（档位是全局的，不会随材质实例变化），因此**不会像 Static Switch 那样随材质实例爆炸**。

---

## 8. 材质属性与 `MP_*` 枚举

translator 按材质属性（Material Property）逐个生成代码。常用属性：

| 枚举 | 对应材质引脚 |
|------|-------------|
| `MP_BaseColor` | Base Color |
| `MP_Metallic` | Metallic |
| `MP_Specular` | Specular |
| `MP_Roughness` | Roughness |
| `MP_EmissiveColor` | Emissive Color |
| `MP_Normal` | Normal |
| `MP_Opacity` | Opacity（半透明） |
| `MP_OpacityMask` | Opacity Mask（Masked） |
| `MP_WorldPositionOffset` | World Position Offset（顶点位移） |
| `MP_AmbientOcclusion` | Ambient Occlusion |
| `MP_SubsurfaceColor` | Subsurface Color |
| `MP_Refraction` | Refraction |
| `MP_CustomData0` / `MP_CustomData1` | 自定义数据（自定义 Shading Model 用） |
| `MP_ShadingModelFromMaterialExpression` | 材质表达式决定着色模型 |

> 这些枚举在 `EngineTypes.h` 的 `EMaterialProperty` 中定义，用于 C++ 侧查询材质编译输出（例如判断 `MP_WorldPositionOffset` 是否被连接）。

---

## 9. 常见坑清单

| # | 现象 | 根因 | 解法 |
|---|------|------|------|
| 1 | 改一个材质就卡很久 | 该材质被很多实例引用 / 变体多 | 减少 Static Switch |
| 2 | 运行时卡顿（shader 编译） | 动态改了 Static Parameter | 改用 MID + 普通参数 / MPC |
| 3 | 材质函数改一行，全项目重编 | 内联展开 + DDC key 变化 | 谨慎修改；用版本化新函数 |
| 4 | Custom 节点编译报错行号对不上 | 报错在生成代码里 | 开 `r.DumpShaderDebugInfo` 看生成结果 |
| 5 | 材质参数改了没反应 | 参数名拼错 / 用了静态参数名 | 检查大小写 |
| 6 | MPC 参数读不到 | 未在关卡中创建 CollectionInstance | 用 `GetParameterCollectionInstance` 创建 |
| 7 | 采样器数量超限 | 材质里纹理过多 | 合并为图集 / 用共享采样器 |
| 8 | 半透明材质排序错乱 | Translucency 本身的限制 | 调整排序优先级或用 OIT 方案 |
| 9 | 移动端材质行为不同 | Feature Level 差异 | 用 Feature Level Switch |
| 10 | 材质在 Cook 后缺失某些效果 | 变体未被 Cook | 检查 Cook 日志与 PSO 记录 |

---

## 10. 小结

| 主题 | 要点 |
|------|------|
| 两个世界 | `UMaterial`（编辑）↔ `FMaterialResource`（渲染，每 FeatureLevel 一份） |
| 代码生成 | `FHLSLMaterialTranslator` 走节点图 → 填 `MaterialTemplate.ush` 的 `GetMaterial*()` |
| 自定义 shader 读材质 | `MakeInitializedMaterialPixelParameters` → `CalcMaterialParameters` → `GetMaterial*()` |
| 参数传递 | 标量/向量打包进 `Material` UB；纹理绑槽位 |
| MID | 运行期改参数，零重编译 |
| MPC | 跨材质全局参数，uniform 级，不产生变体 ⭐ |
| Custom 节点 | 直接写 HLSL；注意 VS/PS 双份、不能用 `#include`、用 `MaterialFloat` |
| 材质函数 | 内联展开；改一个会触发所有引用者重编 |
| Static Switch | 编译期分支 → **变体翻倍**，主要的性能与编译时间风险源 |

---

*上一篇：[06-常用宏速查大全](06-常用宏速查大全.md) | 下一篇：[08-调试、优化与最佳实践](08-调试优化与最佳实践.md)*
