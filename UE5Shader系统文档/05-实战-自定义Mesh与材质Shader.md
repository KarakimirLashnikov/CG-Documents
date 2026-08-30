# 实战：自定义 Mesh 与材质 Shader

> 系列第 **5** 篇。当你的 Pass 需要**读取材质的节点输出**（BaseColor、Roughness、自定义输出）并**绘制网格**时，`FGlobalShader` 就不够了——需要 `FMeshMaterialShader` + 顶点工厂 + MeshPassProcessor。本文给出完整骨架与轻量化替代方案。

---

## 1. 什么时候必须用 `FMeshMaterialShader`

| 需求 | 用什么 |
|------|-------|
| 只做屏幕空间处理（后处理、全屏、Compute） | `FGlobalShader` |
| 屏幕空间 + 需要材质输出（后处理材质、Light Function） | `FMaterialShader` |
| **绘制网格 + 需要材质输出** | `FMeshMaterialShader` |
| 绘制网格 + 不需要材质（线框、调试绘制） | `FGlobalShader` + 手动绑 VS/PS 也行 |

典型场景：

- 自定义 Shading Model 的 BasePass 分支
- 自定义的 PrePass / 深度 Pass / ID Buffer Pass
- 需要材质参与的特殊 Pass（比如把材质输出烘焙到一张 RT）
- 自定义几何管线（如毛发、地形特殊路径）

---

## 2. 顶点工厂（Vertex Factory）速览

**顶点工厂**决定"顶点数据如何从缓冲送到 shader 的输入语义"。它是 `FMeshMaterialShader` 变体的一个维度。

| 顶点工厂 | 用途 |
|---------|------|
| `FLocalVertexFactory` | 普通静态网格（最常用） |
| `FGPUSkinVertexFactory` | GPU 蒙皮骨骼网格 |
| `FInstancedStaticMeshVertexFactory` | 实例化静态网格 |
| `FLandscapeVertexFactory` | 地形 |
| `FNiagaraVertexFactory` / `FParticleVertexFactory` | 粒子 |
| `FPointCloudVertexFactory` | 点云 |

```
                  ┌─────────────────────┐
                  │   顶点缓冲（VB）     │
                  └──────────┬──────────┘
                             │ 由 VF 声明的顶点流/格式读取
                             ▼
              ┌──────────────────────────────┐
              │  FVertexFactoryInput（HLSL）  │
              │  Position / Normal / UV / ... │
              └──────────────┬───────────────┘
                             │ VF 宏展开 GetVertexFactoryIntermediates 等
                             ▼
                        你的 VS 主函数
```

**关键后果**：同一份 shader 代码要为每个 VF 编译一遍（因为它改变了顶点输入结构）。这也是变体数量的主要来源之一。

---

## 3. 完整骨架

### 3.1 USF

```hlsl
// Shaders/Private/MyMeshPass.usf

#include "/Engine/Public/Platform.ush"
#include "/Engine/Private/Common.ush"
#include "/Engine/Private/VertexFactoryCommon.ush"

// ★ 顶点工厂头文件由引擎在编译时注入（不同 VF 对应不同文件）
// 引擎会为 FLocalVertexFactory 等自动 #include 对应的 .ush

struct FMyMeshPassVSToPS
{
    float4 Position      : SV_POSITION;
    float2 UV            : TEXCOORD0;
    float3 WorldNormal   : TEXCOORD1;
    float3 BaseColor     : TEXCOORD2;    // 从材质取，插值给 PS
#if USE_CUSTOM_DATA
    float2 CustomData    : TEXCOORD3;
#endif
};

// ---------------- 顶点着色器 ----------------
void MainVS(
    FVertexFactoryInput Input,
    out FMyMeshPassVSToPS Output)
{
    // ① 由顶点工厂提供的中间数据
    FVertexFactoryIntermediates VFIntermediates = GetVertexFactoryIntermediates(Input);

    float3 WorldPosition = VertexFactoryGetWorldPosition(Input, VFIntermediates);
    float3 WorldNormal   = VertexFactoryGetWorldNormal(Input, VFIntermediates);
    float2 UV            = VertexFactoryGetUV(Input, VFIntermediates, 0);

    // ② 世界位置偏移（材质里的 World Position Offset 在这里生效）
    //    需要用材质系统提供的辅助函数，见 07 篇
    // WorldPosition += GetMaterialWorldPositionOffset(...);

    // ③ 变换到裁剪空间（用 View 的矩阵）
    Output.Position    = mul(float4(WorldPosition, 1.0f), View.TranslatedWorldToClip);
    Output.UV          = UV;
    Output.WorldNormal = WorldNormal;

    // ④ 从材质取 BaseColor（顶点着色器版本的材质查询）
    Output.BaseColor   = Material.vtFetch?...;   // 见 §3.3 说明

#if USE_CUSTOM_DATA
    Output.CustomData  = UV * 2.0f;
#endif
}

// ---------------- 像素着色器 ----------------
void MainPS(
    FMyMeshPassVSToPS Input,
    out float4 OutColor : SV_Target0)
{
    float3 N = normalize(Input.WorldNormal);
    float  NdL = saturate(dot(N, float3(0.0f, 0.0f, 1.0f)));

    float3 Color = Input.BaseColor * (0.2f + 0.8f * NdL);

    OutColor = float4(Color, 1.0f);
}
```

> ⚠️ 上面的 `FVertexFactoryInput` / `GetVertexFactoryIntermediates` / `VertexFactoryGetWorldPosition` 等**由顶点工厂的 HLSL 头文件提供**（如 `/Engine/Private/LocalVertexFactory.ush`）。引擎在编译材质着色器时会自动注入对应的 VF 头文件，你**不需要也不能**手动 include 具体某个 VF。

### 3.2 C++ Shader 类

```cpp
// Public/MyMeshPassShaders.h
#pragma once
#include "MeshMaterialShader.h"
#include "ShaderParameterStruct.h"

// ---------------- 顶点着色器 ----------------
class FMyMeshPassVS : public FMeshMaterialShader
{
public:
    DECLARE_SHADER_TYPE(FMyMeshPassVS, MeshMaterial);

    // ---- Permutation ----
    class FUseCustomDataDim : SHADER_PERMUTATION_BOOL("USE_CUSTOM_DATA");
    using FPermutationDomain = TShaderPermutationDomain<FUseCustomDataDim>;

    FMyMeshPassVS() {}
    FMyMeshPassVS(const FMeshMaterialShaderType::CompiledShaderInitializerType& Initializer)
        : FMeshMaterialShader(Initializer)
    {
        // ★ 绑定 PassUniformBuffer（用于访问 SceneTextures 等全局 UB）
        PassUniformBuffer.Bind(Initializer.ParameterMap,
                               FSceneTexturesUniformBuffer::StaticStructMetadata.GetShaderVariableName());
    }

    static bool ShouldCompilePermutation(const FMeshMaterialShaderPermutationParameters& Parameters)
    {
        // 只在不透明材质上编译
        if (IsTranslucentBlendMode(Parameters.MaterialParameters.BlendMode))
            return false;
        return true;
    }

    static void ModifyCompilationEnvironment(
        const FMeshMaterialShaderPermutationParameters& Parameters,
        FShaderCompilerEnvironment& OutEnvironment)
    {
        FMeshMaterialShader::ModifyCompilationEnvironment(Parameters, OutEnvironment);
    }
};

// ---------------- 像素着色器 ----------------
class FMyMeshPassPS : public FMeshMaterialShader
{
public:
    DECLARE_SHADER_TYPE(FMyMeshPassPS, MeshMaterial);

    class FUseCustomDataDim : SHADER_PERMUTATION_BOOL("USE_CUSTOM_DATA");
    using FPermutationDomain = TShaderPermutationDomain<FUseCustomDataDim>;

    FMyMeshPassPS() {}
    FMyMeshPassPS(const FMeshMaterialShaderType::CompiledShaderInitializerType& Initializer)
        : FMeshMaterialShader(Initializer)
    {
        PassUniformBuffer.Bind(Initializer.ParameterMap,
                               FSceneTexturesUniformBuffer::StaticStructMetadata.GetShaderVariableName());
    }

    static bool ShouldCompilePermutation(const FMeshMaterialShaderPermutationParameters& Parameters)
    {
        return !IsTranslucentBlendMode(Parameters.MaterialParameters.BlendMode);
    }
};
```

```cpp
// Private/MyMeshPassShaders.cpp
#include "MyMeshPassShaders.h"

IMPLEMENT_MATERIAL_SHADER_TYPE(, FMyMeshPassVS,
    TEXT("/Plugin/MyPlugin/Private/MyMeshPass.usf"), TEXT("MainVS"), SF_Vertex);

IMPLEMENT_MATERIAL_SHADER_TYPE(, FMyMeshPassPS,
    TEXT("/Plugin/MyPlugin/Private/MyMeshPass.usf"), TEXT("MainPS"), SF_Pixel);
```

### 3.3 在 shader 里读材质输出

材质系统通过 `MaterialTemplate.ush` 提供了一组查询函数。**在像素着色器里**的标准写法：

```hlsl
#include "/Engine/Private/MaterialTemplate.ush"   // 材质着色器由引擎自动包含

void MainPS(...)
{
    // ① 组装材质参数
    FMaterialPixelParameters MaterialParameters = MakeInitializedMaterialPixelParameters(Input, SvPosition);

    // ② 计算材质输入（会执行材质节点图生成的 HLSL）
    FPixelMaterialInputs PixelMaterialInputs;
    CalcMaterialParameters(MaterialParameters, PixelMaterialInputs, SvPosition, bIsFrontFace);

    // ③ 查询各属性
    float3 BaseColor = GetMaterialBaseColor(PixelMaterialInputs);
    float  Metallic  = GetMaterialMetallic(PixelMaterialInputs);
    float  Roughness = GetMaterialRoughness(PixelMaterialInputs);
    float3 Emissive  = GetMaterialEmissive(PixelMaterialInputs);
    float  Opacity   = GetMaterialOpacity(PixelMaterialInputs);
    float3 Normal    = GetMaterialNormal(MaterialParameters, PixelMaterialInputs);
}
```

| 阶段 | 结构体 | 用途 |
|------|--------|------|
| 顶点 | `FMaterialVertexParameters` | VS 侧查询（如 WPO） |
| 像素 | `FMaterialPixelParameters` + `FPixelMaterialInputs` | PS 侧查询全部材质属性 |

> **VS 里读 BaseColor 通常不可用**：多数材质连接在像素着色阶段才有意义。若确实需要，材质会在 VS 里生成对应代码（通过 `MATERIAL_OUTPUT_*_VS` 之类的路径），但成本较高。**实践中：VS 只处理 WPO / 位移，颜色一律在 PS 里读**。

---

## 4. MeshPassProcessor：把 MeshBatch 变成绘制命令

`FMeshPassProcessor` 的职责：遍历可见的 `FMeshBatch` → 查材质 shader → 填充绑定 → 生成 `FMeshDrawCommand`。

```
可见集（FScene 的 Primitive 列表）
    │
    │  FMeshPassProcessor::AddMeshBatch(...)
    ▼
┌────────────────────────────────────────────┐
│ TryAddMeshBatch                             │
│  · 取材质（MaterialRenderProxy → FMaterial）│
│  · 取顶点工厂类型                             │
│  · 构造 PermutationDomain                    │
│  · 查 shader：GetShader<T>(VFType, Perm)     │
│  · 填充 FMeshDrawSingleShaderBindings        │
└──────────────────┬─────────────────────────┘
                   │ BuildMeshDrawCommands(...)
                   ▼
            FMeshDrawCommand（缓存 / 每帧生成）
```

### 4.1 骨架

```cpp
// Public/MyMeshPassProcessor.h
#pragma once
#include "MeshPassProcessor.h"

class FMyMeshPassProcessor : public FMeshPassProcessor
{
public:
    FMyMeshPassProcessor(
        const FScene* Scene,
        const FSceneView* InViewIfDynamicMeshCommand,
        const FMeshPassProcessorRenderState& InPassDrawRenderState,
        FMeshPassDrawListContext* InDrawListContext);

    // ★ 必须实现的虚函数
    virtual void AddMeshBatch(
        const FMeshBatch& RESTRICT MeshBatch,
        uint64 BatchElementMask,
        const FPrimitiveSceneProxy* RESTRICT PrimitiveSceneProxy,
        int32 StaticMeshId = -1) override final;

private:
    bool TryAddMeshBatch(
        const FMeshBatch& RESTRICT MeshBatch,
        uint64 BatchElementMask,
        const FPrimitiveSceneProxy* RESTRICT PrimitiveSceneProxy,
        int32 StaticMeshId,
        const FMaterialRenderProxy& MaterialRenderProxy,
        const FMaterial& Material);

    bool Process(
        const FMeshBatch& MeshBatch,
        uint64 BatchElementMask,
        int32 StaticMeshId,
        const FPrimitiveSceneProxy* PrimitiveSceneProxy,
        const FMaterialRenderProxy& MaterialRenderProxy,
        const FMaterial& Material);

    FMeshPassProcessorRenderState PassDrawRenderState;
};
```

### 4.2 实现

```cpp
// Private/MyMeshPassProcessor.cpp
#include "MyMeshPassProcessor.h"
#include "MyMeshPassShaders.h"
#include "ScenePrivate.h"

FMyMeshPassProcessor::FMyMeshPassProcessor(
    const FScene* Scene,
    const FSceneView* InViewIfDynamicMeshCommand,
    const FMeshPassProcessorRenderState& InPassDrawRenderState,
    FMeshPassDrawListContext* InDrawListContext)
    : FMeshPassProcessor(Scene, Scene->GetFeatureLevel(), InViewIfDynamicMeshCommand, InDrawListContext)
    , PassDrawRenderState(InPassDrawRenderState)
{
}

void FMyMeshPassProcessor::AddMeshBatch(
    const FMeshBatch& RESTRICT MeshBatch,
    uint64 BatchElementMask,
    const FPrimitiveSceneProxy* RESTRICT PrimitiveSceneProxy,
    int32 StaticMeshId)
{
    // ① 拿到材质代理
    const FMaterialRenderProxy* FallbackMaterialRenderProxyPtr = nullptr;
    const FMaterial& Material = MeshBatch.MaterialRenderProxy
        ->GetMaterialWithFallback(FeatureLevel, FallbackMaterialRenderProxyPtr);

    const FMaterialRenderProxy& MaterialRenderProxy =
        FallbackMaterialRenderProxyPtr ? *FallbackMaterialRenderProxyPtr : *MeshBatch.MaterialRenderProxy;

    // ② 走裁剪与处理
    if (!TryAddMeshBatch(MeshBatch, BatchElementMask, PrimitiveSceneProxy,
                         StaticMeshId, MaterialRenderProxy, Material))
    {
        // 失败时用默认材质兜底
        const FMaterial& DefaultMaterial =
            UMaterial::GetDefaultMaterial(MD_Surface)->GetRenderProxy()->GetMaterialWithFallback(FeatureLevel, ...);
        TryAddMeshBatch(MeshBatch, BatchElementMask, PrimitiveSceneProxy,
                        StaticMeshId, *UMaterial::GetDefaultMaterial(MD_Surface)->GetRenderProxy(), DefaultMaterial);
    }
}

bool FMyMeshPassProcessor::TryAddMeshBatch(
    const FMeshBatch& RESTRICT MeshBatch,
    uint64 BatchElementMask,
    const FPrimitiveSceneProxy* RESTRICT PrimitiveSceneProxy,
    int32 StaticMeshId,
    const FMaterialRenderProxy& MaterialRenderProxy,
    const FMaterial& Material)
{
    // 材质是否参与这个 Pass
    if (!Material.GetRenderingThreadShaderMap()) return false;
    if (Material.IsTranslucentBlendMode(Material.GetBlendMode())) return false;

    return Process(MeshBatch, BatchElementMask, StaticMeshId,
                   PrimitiveSceneProxy, MaterialRenderProxy, Material);
}

bool FMyMeshPassProcessor::Process(
    const FMeshBatch& MeshBatch,
    uint64 BatchElementMask,
    int32 StaticMeshId,
    const FPrimitiveSceneProxy* PrimitiveSceneProxy,
    const FMaterialRenderProxy& MaterialRenderProxy,
    const FMaterial& Material)
{
    const FVertexFactory* VertexFactory = MeshBatch.VertexFactory;
    const FVertexFactoryType* VFType = VertexFactory->GetType();

    // ① 构造 Permutation
    FMyMeshPassVS::FPermutationDomain VSPermutation;
    FMyMeshPassPS::FPermutationDomain PSPermutation;
    VSPermutation.Set<FMyMeshPassVS::FUseCustomDataDim>(false);
    PSPermutation.Set<FMyMeshPassPS::FUseCustomDataDim>(false);

    // ② 查 shader（按 材质 + VF + Permutation）
    TMeshProcessorShaders<FMyMeshPassVS, FMyMeshPassPS> Shaders;
    Shaders.VertexShader = Material.GetShaderMap()->GetShader<FMyMeshPassVS>(VFType, VSPermutation);
    Shaders.PixelShader  = Material.GetShaderMap()->GetShader<FMyMeshPassPS>(VFType, PSPermutation);

    if (Shaders.VertexShader.IsNull() || Shaders.PixelShader.IsNull())
        return false;

    // ③ 渲染状态（混合、深度、模板）
    FMeshPassProcessorRenderState DrawRenderState(PassDrawRenderState);
    DrawRenderState.SetBlendState(TStaticBlendState<>::GetRHI());
    DrawRenderState.SetDepthStencilState(
        TStaticDepthStencilState<true, CF_DepthNearOrEqual>::GetRHI());

    // ④ 逐元素绑定
    FMeshMaterialShaderElementData ShaderElementData;
    ShaderElementData.InitializeMeshMaterialData(ViewIfDynamicMeshCommand, PrimitiveSceneProxy, MeshBatch, StaticMeshId, true);

    // ⑤ 生成绘制命令
    const FMeshDrawCommandSortKey SortKey = CalculateMeshStaticSortKey(*Shaders.VertexShader, *Shaders.PixelShader);

    BuildMeshDrawCommands(
        MeshBatch,
        BatchElementMask,
        PrimitiveSceneProxy,
        MaterialRenderProxy,
        Material,
        DrawRenderState,
        Shaders,
        MeshBatch.Type == PT_LineList ? FM_Solid : MeshBatch.Type == PT_TriangleList ? FM_Solid : FM_Solid,
        CM_CW,                                   // Cull Mode
        SortKey,
        EMeshPassFeatures::Default,
        ShaderElementData);

    return true;
}
```

> ⚠️ `BuildMeshDrawCommands`、`FMeshMaterialShaderElementData::InitializeMeshMaterialData`、`CalculateMeshStaticSortKey` 的**确切签名在 UE 5.x 小版本间有调整**。最稳妥的做法是**对照引擎里 `DepthRendering.cpp` 的 `FDepthPassMeshProcessor::Process` 抄**，那是最标准的参考实现。

### 4.3 自定义 ShaderBindings

若你的 shader 有自定义参数，需要重写 `GetShaderBindings`：

```cpp
// 在 FMyMeshPassPS 中
virtual void GetShaderBindings(
    const FScene* Scene,
    ERHIFeatureLevel::Type FeatureLevel,
    const FPrimitiveSceneProxy* PrimitiveSceneProxy,
    const FMaterialRenderProxy* MaterialRenderProxy,
    const FMaterial& Material,
    const FMeshPassProcessorRenderState& DrawRenderState,
    const FMeshMaterialShaderElementData& ShaderElementData,
    FMeshDrawSingleShaderBindings& ShaderBindings) const override
{
    // ★ 必须先调基类，它会绑定 PassUniformBuffer 与材质相关参数
    FMeshMaterialShader::GetShaderBindings(
        Scene, FeatureLevel, PrimitiveSceneProxy, MaterialRenderProxy,
        Material, DrawRenderState, ShaderElementData, ShaderBindings);

    // 追加自己的参数
    ShaderBindings.Add(MyCustomParameter, FVector4f(1, 0, 0, 1));
}
```

其中 `MyCustomParameter` 用传统方式声明：

```cpp
LAYOUT_FIELD(FShaderParameter, MyCustomParameter);

// 构造函数里
MyCustomParameter.Bind(Initializer.ParameterMap, TEXT("MyCustomParameter"));
```

---

## 5. 轻量化方案：`DrawDynamicMeshPass`

添加一个全新的 `EMeshPass` 需要改引擎源码（枚举、`GetMeshPassName`、渲染器调度）。**如果只是想画一批网格**，UE5 提供了更轻的入口：

```cpp
#include "MeshPassProcessor.h"

// 在渲染函数（如 ViewExtension 的 PostRenderBasePass）中
DrawDynamicMeshPass(
    View,
    GraphBuilder,
    PassParameters,                    // 你的 RDG 参数（含 RenderTargets）
    [&](FDynamicPassMeshDrawListContext* DynamicMeshPassContext)
    {
        FMyMeshPassProcessor Processor(
            Scene, &View, DrawRenderState, DynamicMeshPassContext);

        // 遍历要绘制的 MeshBatch
        for (const FMeshBatch& MeshBatch : MyMeshBatchList)
        {
            Processor.AddMeshBatch(
                MeshBatch,
                /* BatchElementMask = */ ~0ull,
                MeshBatch.PrimitiveSceneProxy);
        }
    });
```

| 方案 | 侵入性 | 缓存 | 适用 |
|------|-------|------|------|
| `DrawDynamicMeshPass` | 低（不改引擎） | 每帧重建命令 | 动态/少量物体、原型开发 |
| 新 `EMeshPass` + 静态绘制列表 | 高（改引擎） | 命令可缓存，性能好 | 正式管线上量 |

> **推荐路径**：先用 `DrawDynamicMeshPass` 把功能跑通 → 确认性能是瓶颈后，再考虑加 `EMeshPass` 走静态缓存路径。

---

## 6. 变体数量的现实

自定义 Mesh Shader 的变体数：

```
变体数 = 材质数 × 顶点工厂类型数 × Permutation 维度数 × FeatureLevel × 平台
```

假设项目有 500 个材质、用到了 6 种 VF、你的 shader 有 4 个 Permutation 取值：

```
500 × 6 × 4 = 12,000 个变体（每个 FeatureLevel / 平台再乘一遍）
```

**必须做的裁剪**：

```cpp
static bool ShouldCompilePermutation(const FMeshMaterialShaderPermutationParameters& Parameters)
{
    // ① 按 VF 裁剪：只关心静态网格与实例化网格
    const FString VFName = Parameters.VertexFactoryType->GetName();
    if (VFName != TEXT("FLocalVertexFactory") &&
        VFName != TEXT("FInstancedStaticMeshVertexFactory"))
        return false;

    // ② 按材质属性裁剪
    if (IsTranslucentBlendMode(Parameters.MaterialParameters.BlendMode)) return false;
    if (Parameters.MaterialParameters.ShadingModels.IsUnlit()) return false;

    // ③ 按平台裁剪
    if (!IsFeatureLevelSupported(Parameters.Platform, ERHIFeatureLevel::SM5)) return false;

    return true;
}
```

裁剪后可能只剩 **500 × 2 × 4 = 4,000**，再配合 DDC，编译量就可控了。

---

## 7. 常见坑清单

| # | 现象 | 根因 | 解法 |
|---|------|------|------|
| 1 | `GetShader<T>()` 返回空 | 材质未编译完 / Permutation 被裁剪 | 检查 `Material.IsComplete()`；查裁剪条件 |
| 2 | 顶点位置全在原点 | 未用 `VertexFactoryGetWorldPosition` | 走 VF 的辅助函数，别直接读 `Input.Position` |
| 3 | 编译报找不到 `FVertexFactoryInput` | 手写 include 了某个 VF 头文件 | 删掉，引擎自动注入 |
| 4 | 材质参数读不到 | 未调 `CalcMaterialParameters` | 按 §3.3 的标准三步走 |
| 5 | `PassUniformBuffer` 为 null | 构造函数里未 Bind | 补 `PassUniformBuffer.Bind(...)` |
| 6 | 网格不可见但无报错 | Cull Mode / 深度状态配置错误 | 检查 `SetDepthStencilState` |
| 7 | 变体数暴涨 | 未做 VF 与材质属性裁剪 | `ShouldCompilePermutation` |
| 8 | 自定义参数无效 | 未在 `GetShaderBindings` 里 Add | 补 `ShaderBindings.Add(...)` |
| 9 | 动态 Pass 每帧卡顿 | `DrawDynamicMeshPass` 每帧重建命令 | 改用静态 `EMeshPass` |
| 10 | 打包后缺变体 | 材质未 Cook / Permutation 未覆盖 | 检查 Cook 日志与 PSO 记录 |

---

## 8. 小结

| 主题 | 要点 |
|------|------|
| 何时用 | 绘制网格 + 需要材质输出 |
| Shader 类 | `FMeshMaterialShader` + `DECLARE_SHADER_TYPE(..., MeshMaterial)` + `IMPLEMENT_MATERIAL_SHADER_TYPE` |
| 顶点数据 | 通过 VF 的 HLSL 辅助函数获取，**不要手动 include VF 头文件** |
| 材质输出 | `MakeInitializedMaterialPixelParameters` → `CalcMaterialParameters` → `GetMaterial*()` |
| 绘制 | `FMeshPassProcessor::AddMeshBatch` → `BuildMeshDrawCommands` |
| 轻量方案 | `DrawDynamicMeshPass`（不改引擎，原型首选） |
| 变体治理 | 按 VF + 材质属性 + 平台三重裁剪 |

---

*上一篇：[04-实战：自定义 Global Shader](04-实战-自定义GlobalShader.md) | 下一篇：[06-常用宏速查大全](06-常用宏速查大全.md)*
