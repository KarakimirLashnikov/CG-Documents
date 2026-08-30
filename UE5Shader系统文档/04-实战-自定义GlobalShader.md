# 实战：自定义 Global Shader

> 系列第 **4** 篇。这是可以**直接照抄**的一篇：从目录结构、Build.cs、USF 源码、C++ Shader 类、RDG Pass 到挂载进渲染管线，给出三个完整可运行的模板（全屏后处理 / Compute / 带 Permutation）。

---

## 1. 项目结构

推荐做成**插件**，shader 与代码放在一起，便于复用：

```
MyPlugin/
├── MyPlugin.uplugin
├── Shaders/
│   └── Private/
│       ├── MyPostProcess.usf       ← 全屏后处理
│       ├── MyCompute.usf           ← Compute
│       └── MyCommon.ush            ← 公共头（可选）
└── Source/
    └── MyPlugin/
        ├── MyPlugin.Build.cs
        ├── Public/
        │   ├── MyPluginModule.h
        │   ├── MyPostProcess.h
        │   └── MyCompute.h
        └── Private/
            ├── MyPluginModule.cpp   ← 注册虚拟路径 + ViewExtension
            ├── MyPostProcess.cpp
            └── MyCompute.cpp
```

### 1.1 Build.cs

```csharp
using UnrealBuildTool;

public class MyPlugin : ModuleRules
{
    public MyPlugin(ReadOnlyTargetRules Target) : base(Target)
    {
        PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;

        PublicDependencyModuleNames.AddRange(new string[]
        {
            "Core",
            "CoreUObject",
            "Engine",
            "Projects",
        });

        PrivateDependencyModuleNames.AddRange(new string[]
        {
            "RenderCore",     // FGlobalShader / FShaderParameterStruct
            "Renderer",       // FSceneViewExtension / ScreenPass / SceneTextures
            "RHI",            // FRHICommandList / FRDGTexture
            "RenderGraph",    // RDG（部分版本已合入 RenderCore）
        });
    }
}
```

### 1.2 模块实现：注册虚拟路径

```cpp
// MyPluginModule.cpp
#include "MyPluginModule.h"
#include "Interfaces/IPluginManager.h"
#include "ShaderCore.h"
#include "SceneViewExtension.h"

IMPLEMENT_MODULE(FMyPluginModule, MyPlugin);

void FMyPluginModule::StartupModule()
{
    // ★ 关键：让 /Plugin/MyPlugin 虚拟路径指向插件的 Shaders 目录
    const FString PluginShaderDir = FPaths::Combine(
        IPluginManager::Get().FindPlugin(TEXT("MyPlugin"))->GetBaseDir(),
        TEXT("Shaders"));
    AddShaderSourceDirectoryMapping(TEXT("/Plugin/MyPlugin"), PluginShaderDir);

    // 注册 ViewExtension（用于挂载 Pass）
    MyViewExtension = FSceneViewExtensions::NewExtension<FMyViewExtension>();
}

void FMyPluginModule::ShutdownModule()
{
    MyViewExtension.Reset();
}
```

---

## 2. 模板 A：全屏后处理 Pass

### 2.1 USF 源码

```hlsl
// Shaders/Private/MyPostProcess.usf

#include "/Engine/Public/Platform.ush"
#include "/Engine/Private/Common.ush"
#include "/Engine/Private/ScreenPass.ush"

// ---- 参数（与 C++ 参数结构体字段一一对应）----
Texture2D    InputTexture;
SamplerState InputSampler;

float4 TintColor;
float  Intensity;
uint   bEnableVignette;

// ---- 顶点着色器（全屏三角形）----
void MainVS(
    in  float4 InPosition : ATTRIBUTE0,
    in  float2 InUV       : ATTRIBUTE1,
    out float4 OutPosition: SV_POSITION,
    out float2 OutUV      : TEXCOORD0)
{
    OutPosition = InPosition;    // 已经是裁剪空间
    OutUV       = InUV;
}

// ---- 像素着色器 ----
void MainPS(
    in  float4 SvPosition : SV_POSITION,
    in  float2 UV         : TEXCOORD0,
    out float4 OutColor   : SV_Target0)
{
    // ★ 用 UE 的跨平台采样宏，不要直接写 .Sample()
    float3 Color = Texture2DSample(InputTexture, InputSampler, UV).rgb;

    Color *= TintColor.rgb;
    Color  = lerp(Color, Color * Intensity, 0.5f);

    if (bEnableVignette)
    {
        float2 Center = UV - 0.5f;
        float  Vignette = saturate(1.0f - dot(Center, Center) * 1.2f);
        Color *= Vignette;
    }

    OutColor = float4(Color, 1.0f);
}
```

### 2.2 C++ Shader 类

```cpp
// Public/MyPostProcess.h
#pragma once
#include "GlobalShader.h"
#include "ShaderParameterStruct.h"
#include "ScreenPass.h"

class FMyPostProcessPS : public FGlobalShader
{
public:
    DECLARE_GLOBAL_SHADER(FMyPostProcessPS);
    SHADER_USE_PARAMETER_STRUCT(FMyPostProcessPS, FGlobalShader);

    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
        SHADER_PARAMETER_STRUCT_REF(FViewUniformShaderParameters, View)

        SHADER_PARAMETER_TEXTURE(Texture2D,    InputTexture)
        SHADER_PARAMETER_SAMPLER(SamplerState, InputSampler)

        SHADER_PARAMETER(FVector4f, TintColor)
        SHADER_PARAMETER(float,     Intensity)
        SHADER_PARAMETER(uint32,    bEnableVignette)

        RENDER_TARGET_BINDING_SLOTS()     // ★ 全屏 Pass 必备
    END_SHADER_PARAMETER_STRUCT()

    static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters& Parameters)
    {
        // 移动端不需要这个 Pass
        return IsFeatureLevelSupported(Parameters.Platform, ERHIFeatureLevel::SM5);
    }
};
```

```cpp
// Private/MyPostProcess.cpp
#include "MyPostProcess.h"

// 虚拟路径 / 入口函数名 / shader 频率
IMPLEMENT_GLOBAL_SHADER(FMyPostProcessPS,
    "/Plugin/MyPlugin/Private/MyPostProcess.usf",
    "MainPS",
    SF_Pixel);
```

### 2.3 发起 Pass

```cpp
// Private/MyPostProcess.cpp（续）
#include "RenderGraph.h"
#include "PixelShaderUtils.h"

static TAutoConsoleVariable<int32> CVarMyPostProcessEnabled(
    TEXT("r.MyPostProcess.Enabled"), 0,
    TEXT("Enable the custom post process pass."),
    ECVF_RenderThreadSafe);

static TAutoConsoleVariable<float> CVarMyPostProcessIntensity(
    TEXT("r.MyPostProcess.Intensity"), 1.0f,
    TEXT("Intensity of the custom pass."),
    ECVF_RenderThreadSafe);

void AddMyPostProcessPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FRDGTextureRef InputTexture,      // 输入（场景色）
    FRDGTextureRef OutputTexture)     // 输出
{
    if (!CVarMyPostProcessEnabled.GetValueOnRenderThread())
        return;

    RDG_EVENT_SCOPE(GraphBuilder, "MyPostProcess");

    // ① 取 shader
    TShaderMapRef<FMyPostProcessPS> PixelShader(View.ShaderMap);

    // ② 分配并填充参数
    FMyPostProcessPS::FParameters* PassParameters =
        GraphBuilder.AllocParameters<FMyPostProcessPS::FParameters>();

    PassParameters->View          = View.ViewUniformBuffer;
    PassParameters->InputTexture  = InputTexture;
    PassParameters->InputSampler  = TStaticSamplerState<SF_Point>::GetRHI();
    PassParameters->TintColor     = FVector4f(1.0f, 1.0f, 1.0f, 1.0f);
    PassParameters->Intensity     = CVarMyPostProcessIntensity.GetValueOnRenderThread();
    PassParameters->bEnableVignette = 1;

    // ③ 绑定渲染目标
    PassParameters->RenderTargets[0] =
        FRenderTargetBinding(OutputTexture, ERenderTargetLoadAction::ENoAction);

    // ④ 发起全屏 Pass
    FPixelShaderUtils::AddFullscreenPass(
        GraphBuilder,
        View.ShaderMap,
        RDG_EVENT_NAME("MyPostProcess"),
        PixelShader,
        PassParameters,
        View.ViewRect);
}
```

### 2.4 三种全屏 Pass 写法

| 写法 | 适用 | 特点 |
|------|------|------|
| `FPixelShaderUtils::AddFullscreenPass` | 通用 | 内部用全屏三角形，最省事 |
| `AddDrawScreenPass`（`ScreenPass.h`） | 后处理链路内 | 需要 `FScreenPassVS` 顶点着色器；与 `FScreenPassTexture` 体系配合 |
| 手动 `AddPass` + `RHICmdList.DrawPrimitive` | 需要自定义几何时 | 最灵活，但要自己管 VS 与顶点缓冲 |

**`AddDrawScreenPass` 变体**：

```cpp
#include "ScreenPass.h"

TShaderMapRef<FScreenPassVS> VertexShader(View.ShaderMap);
TShaderMapRef<FMyPostProcessPS> PixelShader(View.ShaderMap);

const FScreenPassTextureViewport InputViewport(InputTexture);
const FScreenPassTextureViewport OutputViewport(OutputTexture);

AddDrawScreenPass(
    GraphBuilder,
    RDG_EVENT_NAME("MyPostProcess"),
    View,
    OutputViewport,
    InputViewport,
    PixelShader,
    PassParameters);
```

> 若你在 VS 里需要 UV 计算，`ScreenPass.ush` 提供了 `ScreenPassTextureViewport` 相关辅助；引擎内置的 `FScreenPassVS` 已经把 `ATTRIBUTE0/1` 变换做好，一般不需要自己写 VS。

---

## 3. 模板 B：Compute Shader

### 3.1 USF

```hlsl
// Shaders/Private/MyCompute.usf

#include "/Engine/Public/Platform.ush"
#include "/Engine/Private/Common.ush"

RWTexture2D<float4> OutputTexture;
Texture2D           InputTexture;
SamplerState        InputSampler;

uint2 TextureSize;
float BlurRadius;

// 线程组大小（编译期常量）
#define THREADGROUP_SIZE 8

[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void MainCS(uint3 DispatchThreadId : SV_DispatchThreadID)
{
    uint2 Pixel = DispatchThreadId.xy;

    // ★ 越界检查：dispatch 的线程组数通常向上取整，一定会有多余线程
    if (any(Pixel >= TextureSize))
        return;

    float2 UV = (float2(Pixel) + 0.5f) / float2(TextureSize);
    float2 TexelSize = 1.0f / float2(TextureSize);

    // 简易 3x3 模糊
    float4 Sum = 0;
    for (int y = -1; y <= 1; ++y)
    {
        for (int x = -1; x <= 1; ++x)
        {
            float2 Offset = float2(x, y) * TexelSize * BlurRadius;
            Sum += Texture2DSampleLevel(InputTexture, InputSampler, UV + Offset, 0);
        }
    }

    OutputTexture[Pixel] = Sum / 9.0f;
}
```

### 3.2 C++

```cpp
// Public/MyCompute.h
#pragma once
#include "GlobalShader.h"
#include "ShaderParameterStruct.h"

class FMyComputeCS : public FGlobalShader
{
public:
    DECLARE_GLOBAL_SHADER(FMyComputeCS);
    SHADER_USE_PARAMETER_STRUCT(FMyComputeCS, FGlobalShader);

    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
        // ★ RDG 版本：让 RDG 自动推导 UAV 屏障
        SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2D<float4>, OutputTexture)
        SHADER_PARAMETER_RDG_TEXTURE(Texture2D,               InputTexture)
        SHADER_PARAMETER_SAMPLER(SamplerState,                InputSampler)
        SHADER_PARAMETER(FIntPoint,                           TextureSize)
        SHADER_PARAMETER(float,                               BlurRadius)
    END_SHADER_PARAMETER_STRUCT()

    static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters& Parameters)
    {
        return IsFeatureLevelSupported(Parameters.Platform, ERHIFeatureLevel::SM5);
    }

    // 线程组大小需与 USF 中的 numthreads 一致
    static constexpr int32 ThreadGroupSize = 8;
};
```

```cpp
// Private/MyCompute.cpp
#include "MyCompute.h"
#include "RenderGraph.h"
#include "ComputeShaderUtils.h"   // FComputeShaderUtils::AddPass

IMPLEMENT_GLOBAL_SHADER(FMyComputeCS,
    "/Plugin/MyPlugin/Private/MyCompute.usf",
    "MainCS",
    SF_Compute);

void AddMyComputePass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FRDGTextureRef InputTexture,
    FRDGTextureRef OutputTexture,
    float BlurRadius)
{
    RDG_EVENT_SCOPE(GraphBuilder, "MyCompute");

    const FIntPoint TextureSize = InputTexture->Desc.Extent;

    TShaderMapRef<FMyComputeCS> ComputeShader(View.ShaderMap);

    FMyComputeCS::FParameters* PassParameters =
        GraphBuilder.AllocParameters<FMyComputeCS::FParameters>();

    PassParameters->OutputTexture = GraphBuilder.CreateUAV(OutputTexture);
    PassParameters->InputTexture  = InputTexture;
    PassParameters->InputSampler  = TStaticSamplerState<SF_Bilinear>::GetRHI();
    PassParameters->TextureSize   = TextureSize;
    PassParameters->BlurRadius    = BlurRadius;

    // 组数 = ceil(尺寸 / 线程组大小)
    const FIntVector GroupCount = FComputeShaderUtils::GetGroupCount(
        FIntVector(TextureSize.X, TextureSize.Y, 1),
        FIntVector(FMyComputeCS::ThreadGroupSize, FMyComputeCS::ThreadGroupSize, 1));

    FComputeShaderUtils::AddPass(
        GraphBuilder,
        RDG_EVENT_NAME("MyCompute"),
        ERDGPassFlags::Compute,
        ComputeShader,
        PassParameters,
        GroupCount);
}
```

> ⚠️ **Compute Pass 的三个高频错误**：
> 1. **忘记越界检查** → 越界线程写坏内存
> 2. `numthreads` 与 C++ 的 `ThreadGroupSize` 不一致 → 覆盖不全或重复计算
> 3. 用了 `SHADER_PARAMETER_TEXTURE_UAV`（非 RDG）→ 缺屏障，读写竞态

---

## 4. 模板 C：带 Permutation 的 Shader

### 4.1 C++

```cpp
class FMyToneMapPS : public FGlobalShader
{
public:
    DECLARE_GLOBAL_SHADER(FMyToneMapPS);
    SHADER_USE_PARAMETER_STRUCT(FMyToneMapPS, FGlobalShader);

    // ---- 维度 ----
    class FToneMapOperatorDim : SHADER_PERMUTATION_INT("TONEMAP_OPERATOR", 3); // 0=Reinhard 1=ACES 2=Filmic
    class FUseHistogramDim    : SHADER_PERMUTATION_BOOL("USE_HISTOGRAM");

    using FPermutationDomain = TShaderPermutationDomain<FToneMapOperatorDim, FUseHistogramDim>;

    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
        SHADER_PARAMETER_STRUCT_REF(FViewUniformShaderParameters, View)
        SHADER_PARAMETER_RDG_TEXTURE(Texture2D, InputTexture)
        SHADER_PARAMETER_SAMPLER(SamplerState,  InputSampler)
        SHADER_PARAMETER(float, Exposure)
        RENDER_TARGET_BINDING_SLOTS()
    END_SHADER_PARAMETER_STRUCT()

    // ---- 裁剪 ----
    static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters& Parameters)
    {
        FPermutationDomain PermutationVector(Parameters.PermutationId);

        // Histogram 只在 SM6 上编译
        if (PermutationVector.Get<FUseHistogramDim>() &&
            !IsFeatureLevelSupported(Parameters.Platform, ERHIFeatureLevel::SM6))
            return false;

        return true;
    }

    // ---- 追加宏 ----
    static void ModifyCompilationEnvironment(
        const FGlobalShaderPermutationParameters& Parameters,
        FShaderCompilerEnvironment& OutEnvironment)
    {
        FGlobalShader::ModifyCompilationEnvironment(Parameters, OutEnvironment);
        OutEnvironment.SetDefine(TEXT("MAX_EXPOSURE"), TEXT("16.0f"));
    }
};

IMPLEMENT_GLOBAL_SHADER(FMyToneMapPS,
    "/Plugin/MyPlugin/Private/MyToneMap.usf", "MainPS", SF_Pixel);
```

### 4.2 USF

```hlsl
#include "/Engine/Public/Platform.ush"
#include "/Engine/Private/Common.ush"

Texture2D    InputTexture;
SamplerState InputSampler;
float        Exposure;

float3 Reinhard(float3 C) { return C / (1.0f + C); }
float3 Filmic (float3 C) { return max(0.0f, C - 0.004f) * ...; }
float3 ACES   (float3 C) { ... }

void MainPS(in float4 SvPosition : SV_POSITION, out float4 OutColor : SV_Target0)
{
    float2 UV = SvPositionToBufferUV(SvPosition, View);
    float3 Color = Texture2DSampleLevel(InputTexture, InputSampler, UV, 0).rgb;
    Color *= min(Exposure, MAX_EXPOSURE);

#if TONEMAP_OPERATOR == 0
    Color = Reinhard(Color);
#elif TONEMAP_OPERATOR == 1
    Color = ACES(Color);
#else
    Color = Filmic(Color);
#endif

#if USE_HISTOGRAM
    Color = ApplyHistogramAdjust(Color);
#endif

    OutColor = float4(Color, 1);
}
```

### 4.3 运行时选择

```cpp
// 由 CVar 或画质档决定
const int32 Operator = CVarToneMapOperator.GetValueOnRenderThread();
const bool  bUseHist = CVarUseHistogram.GetValueOnRenderThread();

FMyToneMapPS::FPermutationDomain PermutationVector;
PermutationVector.Set<FMyToneMapPS::FToneMapOperatorDim>(Operator);
PermutationVector.Set<FMyToneMapPS::FUseHistogramDim>(bUseHist);

TShaderMapRef<FMyToneMapPS> PixelShader(View.ShaderMap, PermutationVector);
```

> **性能提示**：Permutation 切换会导致**不同的 PSO**。若每帧在变体间来回切（比如按物体属性切），会破坏批处理。优先用 uniform 分支，只有分支开销真的不可接受时才用 Permutation。

---

## 5. 挂载到渲染管线

### 5.1 ViewExtension（推荐）

```cpp
// Public/MyViewExtension.h
#pragma once
#include "SceneViewExtension.h"

class FMyViewExtension : public FSceneViewExtensionBase
{
public:
    FMyViewExtension(const FAutoRegister& AutoRegister)
        : FSceneViewExtensionBase(AutoRegister) {}

    virtual void SetupViewFamily(FSceneViewFamily& InViewFamily) override {}
    virtual void SetupView(FSceneViewFamily& InViewFamily, FSceneView& InView) override {}
    virtual void BeginRenderViewFamily(FSceneViewFamily& InViewFamily) override {}

    // ★ 在这里加 Pass：整个 ViewFamily 渲染完成后
    virtual void PostRenderViewFamily_RenderThread(
        FRDGBuilder& GraphBuilder, FSceneViewFamily& InViewFamily) override;

    virtual bool IsActiveThisFrame_Internal(const FSceneViewExtensionContext& Context) const override
    {
        return CVarMyPostProcessEnabled.GetValueOnAnyThread() != 0;
    }
};
```

```cpp
// Private/MyViewExtension.cpp
void FMyViewExtension::PostRenderViewFamily_RenderThread(
    FRDGBuilder& GraphBuilder, FSceneViewFamily& InViewFamily)
{
    for (FViewInfo& View : InViewFamily.Views)
    {
        RDG_EVENT_SCOPE_CONDITIONAL(GraphBuilder, InViewFamily.Views.Num() > 1, "View%d", View.StereoPass);

        // 取场景色（版本相关的获取方式，见下）
        FRDGTextureRef SceneColor = ...;
        FRDGTextureRef Output     = ...;

        AddMyPostProcessPass(GraphBuilder, View, SceneColor, Output);
    }
}
```

### 5.2 常见的注入点

| 钩子 | 时机 | 用途 |
|------|------|------|
| `PreRenderViewFamily_RenderThread` | ViewFamily 渲染前 | 准备资源、预计算 |
| `PostRenderBasePass_RenderThread` | BasePass 后 | 需要 GBuffer 的效果 |
| `PrePostProcessPass_RenderThread` | 后处理前 | 后处理前的一次修改 |
| `SubscribeToPostProcessingPass` | 后处理链内的具体阶段后 | 在 Tonemap 后叠加、在 DOF 前注入 |
| `PostRenderViewFamily_RenderThread` | 全部渲染完成后 | 最终叠加、调试绘制 |

**`SubscribeToPostProcessingPass` 示例**：

```cpp
void FMyViewExtension::SubscribeToPostProcessingPass(
    EPostProcessingPass Pass,
    FAfterPassCallbackDelegateArray& InOutPassCallbacks,
    bool bIsPassEnabled)
{
    if (Pass == EPostProcessingPass::Tonemap)
    {
        InOutPassCallbacks.Add(FAfterPassCallbackDelegate::CreateRaw(
            this, &FMyViewExtension::PostProcessPassAfterTonemap_RenderThread));
    }
}

FScreenPassTexture FMyViewExtension::PostProcessPassAfterTonemap_RenderThread(
    FRDGBuilder& GraphBuilder,
    const FSceneView& View,
    const FPostProcessMaterialInputs& InOutInputs)
{
    const FScreenPassTexture& SceneColor =
        InOutInputs.Textures[(uint32)EPostProcessMaterialInput::SceneColor];

    // ... 用 SceneColor.Texture 做输入，创建新纹理作为输出

    return SceneColor;   // 返回替换后的纹理（或原样返回）
}
```

> ⚠️ ViewExtension 与后处理注入的**具体签名在 UE 5.x 小版本间有调整**。以你引擎里 `SceneViewExtension.h` / `PostProcess/PostProcessing.h` 的实际声明为准，本节的思路是通用的。

### 5.3 更"重"的注入方式

| 方式 | 侵入性 | 适用 |
|------|-------|------|
| ViewExtension | 低（不改引擎） | ⭐ 首选 |
| 改 `FDeferredShadingSceneRenderer::Render()` | 高（需改引擎源码） | 需要精确控制 Pass 顺序 |
| 自定义 `FSceneRenderer` 子类 | 最高 | 深度定制渲染路径 |

---

## 6. 从零到跑通的清单

| # | 步骤 | 检查点 |
|---|------|-------|
| 1 | 建插件目录 + `.uplugin` + `.Build.cs` | 编译通过 |
| 2 | `Shaders/Private/xxx.usf` | 文件存在 |
| 3 | `StartupModule` 里 `AddShaderSourceDirectoryMapping` | 无 "Failed to find include file" |
| 4 | C++ Shader 类 + `BEGIN_SHADER_PARAMETER_STRUCT` | 编译通过 |
| 5 | `IMPLEMENT_GLOBAL_SHADER`（路径/入口/频率三者正确） | 编译通过 |
| 6 | 参数结构体字段与 HLSL 变量**逐字对应** | 无"参数全 0" |
| 7 | RDG Pass 填充参数 + `AddFullscreenPass` / `AddPass` | 无崩溃 |
| 8 | ViewExtension 挂载 | `r.MyPass.Enabled 1` 后有效果 |
| 9 | `ShouldCompilePermutation` 裁剪 | 变体数量合理 |
| 10 | 控制台变量开关 | 可运行时开关 |

---

## 7. 排错速查

| 现象 | 原因 | 解法 |
|------|------|------|
| `Failed to find include file "/Plugin/MyPlugin/..."` | 未注册虚拟路径 | `AddShaderSourceDirectoryMapping` |
| 编译通过但画面无变化 | CVar 默认关闭 / ViewExtension 未激活 | 检查 `IsActiveThisFrame_Internal` |
| shader 里参数全 0 | 字段名与 HLSL 变量名不一致 | 逐字比对 |
| 全屏 Pass 输出黑 | `RENDER_TARGET_BINDING_SLOTS()` 缺失或未填 `RenderTargets[0]` | 补上 |
| Compute 结果错乱 | 缺越界检查 / 组数算错 | 补 `if (any(Pixel >= TextureSize)) return;` |
| UAV 读写竞态 | 用了非 RDG 的 UAV 宏 | 改 `SHADER_PARAMETER_RDG_TEXTURE_UAV` |
| 报 `GetShader<T>()` 返回空 | `ShouldCompilePermutation` 裁掉了 | 检查裁剪条件 |
| 改了 USF 没生效 | DDC 缓存 | `recompileshaders changed` |
| 其他模块查不到 shader 类型 | 未用 `EXPORTED` 宏 | 改用 `DECLARE_EXPORTED_SHADER_TYPE` |
| 打包后失效 | shader 未被 Cook | 确认插件在 Cook 范围内、shader 目录被包含 |

---

## 8. 小结

| 模板 | 关键 API | 关键陷阱 |
|------|---------|---------|
| **全屏后处理** | `FPixelShaderUtils::AddFullscreenPass` + `RENDER_TARGET_BINDING_SLOTS()` | 忘绑定 RenderTarget |
| **Compute** | `FComputeShaderUtils::AddPass` + `SHADER_PARAMETER_RDG_TEXTURE_UAV` | 越界检查、组数取整、缺屏障 |
| **Permutation** | `SHADER_PERMUTATION_*` + `ShouldCompilePermutation` | 变体爆炸、每帧切变体 |
| **挂载** | `FSceneViewExtensionBase` + `FSceneViewExtensions::NewExtension<T>()` | 签名随版本变化 |

**下一步**：如果你的 Pass 需要读材质节点的输出、或要绘制网格，继续看 [05 篇：自定义 Mesh 与材质 Shader](05-实战-自定义Mesh与材质Shader.md)。

---

*上一篇：[03-Shader 参数体系](03-Shader参数体系.md) | 下一篇：[05-实战：自定义 Mesh 与材质 Shader](05-实战-自定义Mesh与材质Shader.md)*
