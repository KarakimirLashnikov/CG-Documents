# Shader 参数体系

> 系列第 **3** 篇。UE 的参数传递完全建立在 **Uniform Buffer + 宏生成的元数据** 之上。本文系统讲解 `SHADER_PARAMETER_*` 家族、参数结构体的声明/填充/绑定三段式、全局 Uniform Buffer 的定义与注册、RDG 资源参数，以及 LWC 带来的精度陷阱。

---

## 1. 三层抽象

```
┌────────────────────────────────────────────────────────────┐
│ ① 声明（编译期）                                              │
│    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )               │
│        SHADER_PARAMETER(float, Intensity)                    │
│        SHADER_PARAMETER_TEXTURE(Texture2D, InputTexture)     │
│    END_SHADER_PARAMETER_STRUCT()                             │
│                                                              │
│    → 宏生成 FShaderParametersMetadata（成员名/类型/偏移/大小）   │
│    → 生成 C++ 结构体 FParameters（可直接赋值的 POD）           │
└───────────────────────────┬────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ ② 填充（每帧/每 Pass）                                        │
│    FParameters Parameters;                                   │
│    Parameters.Intensity    = 1.5f;                           │
│    Parameters.InputTexture = SomeTexture;                    │
└───────────────────────────┬────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────┐
│ ③ 绑定（RDG / RHI）                                           │
│    按元数据把字段值拷进 Uniform Buffer 的对应偏移，            │
│    把纹理/Sampler/UAV 绑到对应资源槽位                        │
└────────────────────────────────────────────────────────────┘
```

**核心优势**：HLSL 里的变量名与 C++ 里的字段名**由同一处声明保证一致**（宏同时生成两侧），消除了"手写两边对不上"的问题。

---

## 2. 参数结构体宏全解

### 2.1 骨架

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FMyParameters, )
    // 成员...
END_SHADER_PARAMETER_STRUCT()
```

宏展开后会：

1. 定义 `struct FMyParameters { ... }`
2. 定义 `FMyParameters::FTypeInfo` 与静态元数据
3. 生成静态断言（成员大小、对齐、HLSL 类型匹配）

在 shader 类中使用：

```cpp
class FMyShader : public FGlobalShader
{
public:
    DECLARE_GLOBAL_SHADER(FMyShader);
    SHADER_USE_PARAMETER_STRUCT(FMyShader, FGlobalShader);

    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
        SHADER_PARAMETER(FVector4f, TintColor)
        RENDER_TARGET_BINDING_SLOTS()
    END_SHADER_PARAMETER_STRUCT()
};
```

> `SHADER_USE_PARAMETER_STRUCT(ShaderClass, BaseClass)` 等价于告诉框架："这个 shader 用内部定义的 `FParameters` 作为参数"。

### 2.2 标量 / 向量 / 矩阵

| 宏 | HLSL 对应 | 说明 |
|----|----------|------|
| `SHADER_PARAMETER(Type, Name)` | `Type Name;` | 标量/向量/矩阵/结构体值 |
| `SHADER_PARAMETER_ARRAY(Type, Name, Count)` | `Type Name[Count];` | 定长数组 |

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
    SHADER_PARAMETER(uint32,  FrameIndex)
    SHADER_PARAMETER(float,   Intensity)
    SHADER_PARAMETER(FVector2f, ViewportExtent)
    SHADER_PARAMETER(FVector4f, TintColor)
    SHADER_PARAMETER(FMatrix44f, ViewProjMatrix)
    SHADER_PARAMETER_ARRAY(FVector4f, LightColors, 4)
END_SHADER_PARAMETER_STRUCT()
```

对应 HLSL：

```hlsl
uint   FrameIndex;
float  Intensity;
float2 ViewportExtent;
float4 TintColor;
float4x4 ViewProjMatrix;
float4 LightColors[4];
```

#### ⚠️ LWC 精度陷阱

UE5 引入大世界坐标后，C++ 侧的 `FVector` / `FMatrix` 是 **double**。**shader 侧仍是 float**。

| ❌ 错误 | ✅ 正确 |
|--------|--------|
| `SHADER_PARAMETER(FVector, Pos)` | `SHADER_PARAMETER(FVector3f, Pos)` |
| `SHADER_PARAMETER(FVector4, Color)` | `SHADER_PARAMETER(FVector4f, Color)` |
| `SHADER_PARAMETER(FMatrix, M)` | `SHADER_PARAMETER(FMatrix44f, M)` |
| `SHADER_PARAMETER(FLinearColor, C)` | `SHADER_PARAMETER(FVector4f, C)` |

> 引擎会在编译期报错提示你替换。若确实需要传双精度，UE 提供了 LWC 专用类型，但**绝大多数 shader 参数不该用 double**。

#### ⚠️ 向量对齐陷阱

HLSL 的 struct packing 规则下，`float3` 后面的成员会被对齐到 16 字节边界：

```hlsl
// HLSL
float3 Position;    // offset 0
float  Radius;      // offset 12  ← 正确，紧贴
float3 Normal;      // offset 16  ← 被对齐！
```

```cpp
// C++：必须让布局与 HLSL 一致
SHADER_PARAMETER(FVector3f, Position)   // 12 bytes
SHADER_PARAMETER(float,     Radius)     // 4 bytes  → 凑够 16
SHADER_PARAMETER(FVector3f, Normal)
```

> **安全做法**：能用 `FVector4f` 就用 `FVector4f`，避免手动算对齐。或者用 `SHADER_PARAMETER(FVector3f, X)` 但紧跟着一个 `float` 补齐。
> UE 的宏会生成静态断言，C++ 结构体大小与 HLSL 声明不一致时会**编译报错**——这正是参数结构体的价值。

### 2.3 纹理 / 采样器 / 资源

| 宏 | 用途 | HLSL 声明写法 |
|----|------|--------------|
| `SHADER_PARAMETER_TEXTURE(Type, Name)` | 普通纹理 | `Texture2D Name;` |
| `SHADER_PARAMETER_SRV(Type, Name)` | SRV | `Texture2D Name;` / `StructuredBuffer<T> Name;` |
| `SHADER_PARAMETER_UAV(Type, Name)` | UAV | `RWTexture2D<float4> Name;` |
| `SHADER_PARAMETER_SAMPLER(Type, Name)` | 采样器 | `SamplerState Name;` |

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
    SHADER_PARAMETER_TEXTURE(Texture2D,            InputTexture)
    SHADER_PARAMETER_SAMPLER(SamplerState,         InputSampler)
    SHADER_PARAMETER_SRV(Texture2D<float4>,        InputSRV)
    SHADER_PARAMETER_UAV(RWTexture2D<float4>,      OutputUAV)
    SHADER_PARAMETER_SRV(StructuredBuffer<float4>, DataBuffer)
    SHADER_PARAMETER_UAV(RWStructuredBuffer<float4>, OutDataBuffer)
END_SHADER_PARAMETER_STRUCT()
```

> ⚠️ **HLSL 侧必须用 UE 的跨平台宏声明资源**：
> ```hlsl
> Texture2D InputTexture;              // ❌ 不够跨平台
> TEXTURE2D(InputTexture);             // ✅ 推荐
> SAMPLERSTATE(InputSampler);          // ✅
> RWTEXTURE2D(OutputUAV, float4);      // ✅
> ```
> 详见 [06 篇](06-常用宏速查大全.md)。

### 2.4 渲染目标

```cpp
RENDER_TARGET_BINDING_SLOTS()
```

这是**全屏/后处理 Pass 必备**的宏：它声明了 `RenderTargets[0..7]` 与深度模板的绑定槽位。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
    SHADER_PARAMETER_STRUCT_REF(FViewUniformShaderParameters, View)
    SHADER_PARAMETER_TEXTURE(Texture2D, InputTexture)
    RENDER_TARGET_BINDING_SLOTS()      // ← 必须
END_SHADER_PARAMETER_STRUCT()

// 填充
PassParameters->RenderTargets[0] = FRenderTargetBinding(OutputTexture, ERenderTargetLoadAction::ENoAction);
```

### 2.5 结构体嵌套

| 宏 | 语义 | 何时用 |
|----|------|-------|
| `SHADER_PARAMETER_STRUCT(Type, Name)` | 作为**嵌套成员**（生成一个子 UB 内的成员） | 想让 HLSL 里写 `Params.Nested.Field` |
| `SHADER_PARAMETER_STRUCT_INCLUDE(Type, Name)` | **内联展开**成员到父结构体 | 复用一组参数（如 `FSceneTextureShaderParameters`） |
| `SHADER_PARAMETER_STRUCT_REF(Type, Name)` | 引用一个**全局 Uniform Buffer** | 引用 `View`、`Primitive` 这类全局 UB |

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
    // 引用全局 UB（按名字绑定）
    SHADER_PARAMETER_STRUCT_REF(FViewUniformShaderParameters, View)

    // 内联展开场景纹理参数（成员直接进本结构体）
    SHADER_PARAMETER_STRUCT_INCLUDE(FSceneTextureShaderParameters, SceneTextures)

    // 嵌套子结构体
    SHADER_PARAMETER_STRUCT(FMyNestedParams, Nested)
END_SHADER_PARAMETER_STRUCT()
```

---

## 3. 全局 Uniform Buffer

当你需要"**一份数据被多个 shader / 多个 Pass 共享**"时，用全局 Uniform Buffer。

### 3.1 定义与注册

```cpp
// MyShaderParams.h
#pragma once
#include "ShaderParameterStruct.h"

BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT(FMyViewParams, )
    SHADER_PARAMETER(FMatrix44f, ViewProjMatrix)
    SHADER_PARAMETER(FMatrix44f, InvViewProjMatrix)
    SHADER_PARAMETER(FVector4f,  CameraWorldPos)
    SHADER_PARAMETER(FVector4f,  ScreenSize)
    SHADER_PARAMETER(float,      Time)
END_GLOBAL_SHADER_PARAMETER_STRUCT()
```

```cpp
// MyShaderParams.cpp
#include "MyShaderParams.h"

// 第二个参数是 HLSL 里看到的变量名
IMPLEMENT_GLOBAL_SHADER_PARAMETER_STRUCT(FMyViewParams, "MyView");
```

HLSL 侧直接可用（无需任何声明，全局可见）：

```hlsl
#include "/Engine/Private/Common.ush"

float4 ProjectToClip(float3 WorldPos)
{
    return mul(float4(WorldPos, 1), MyView.ViewProjMatrix);
}
```

### 3.2 更新值

**方式 A：立即模式（每帧一次，最常用）**

```cpp
FMyViewParams Params;
Params.ViewProjMatrix = View.ViewMatrices.GetViewProjMatrix();
Params.CameraWorldPos = FVector4f(View.ViewMatrices.GetViewOrigin());
Params.Time           = View.Family->Time.GetWorldTimeSeconds();

TUniformBufferRef<FMyViewParams> UB =
    TUniformBufferRef<FMyViewParams>::CreateUniformBufferImmediate(
        Params, UniformBuffer_SingleFrame);

// 绑定到 Parameter 结构体
PassParameters->MyView = UB;
```

| `EUniformBufferUsage` | 语义 |
|----------------------|------|
| `UniformBuffer_SingleFrame` | 只在一帧内有效（引擎每帧回收） |
| `UniformBuffer_MultiFrame` | 跨帧有效（需自行管理更新） |
| `UniformBuffer_SingleDraw` | 只在一次 draw 内有效（开销最小，也最少用） |

**方式 B：RDG（推荐，生命周期交给 RDG）**

```cpp
FMyViewParams* Params = GraphBuilder.AllocParameters<FMyViewParams>();
Params->ViewProjMatrix = ...;
TRDGUniformBufferRef<FMyViewParams> UB = GraphBuilder.CreateUniformBuffer(Params);
PassParameters->MyView = UB;
```

> **RDG 方式的好处**：资源引用会被 RDG 正确追踪，避免"UAV 未同步"这类竞态。

### 3.3 在参数结构体中引用

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
    SHADER_PARAMETER_STRUCT_REF(FMyViewParams, MyView)
END_SHADER_PARAMETER_STRUCT()
```

> ⚠️ 被引用的类型必须已用 `IMPLEMENT_GLOBAL_SHADER_PARAMETER_STRUCT` 注册，否则链接期会报找不到符号。

---

## 4. RDG 资源参数

RDG（`FRenderGraphBuilder`）管理的纹理/缓冲有专门的宏，让 RDG 自动处理屏障与生命周期。

| 宏 | 用途 |
|----|------|
| `SHADER_PARAMETER_RDG_TEXTURE(Type, Name)` | RDG 纹理（作为纹理资源） |
| `SHADER_PARAMETER_RDG_TEXTURE_SRV(Type, Name)` | RDG 纹理的 SRV |
| `SHADER_PARAMETER_RDG_TEXTURE_UAV(Type, Name)` | RDG 纹理的 UAV |
| `SHADER_PARAMETER_RDG_BUFFER(Type, Name)` | RDG 缓冲 |
| `SHADER_PARAMETER_RDG_BUFFER_SRV(Type, Name)` | RDG 缓冲的 SRV |
| `SHADER_PARAMETER_RDG_BUFFER_UAV(Type, Name)` | RDG 缓冲的 UAV |
| `SHADER_PARAMETER_RDG_UNIFORM_BUFFER(Type, Name)` | RDG Uniform Buffer |

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D,                 InputTexture)
    SHADER_PARAMETER_RDG_TEXTURE_SRV(Texture2D<float4>,     InputSRV)
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2D<float4>,   OutputUAV)
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<float4>, DataSRV)
    SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<float4>, DataUAV)
    SHADER_PARAMETER_SAMPLER(SamplerState,                  BilinearSampler)
END_SHADER_PARAMETER_STRUCT()
```

填充：

```cpp
FParameters* PassParameters = GraphBuilder.AllocParameters<FParameters>();
PassParameters->InputTexture = InputTexture;                                  // FRDGTextureRef
PassParameters->InputSRV     = GraphBuilder.CreateSRV(InputTexture);          // FRDGTextureSRVRef
PassParameters->OutputUAV    = GraphBuilder.CreateUAV(OutputTexture);         // FRDGTextureUAVRef
PassParameters->BilinearSampler = TStaticSamplerState<SF_Bilinear>::GetRHI();
```

> **RDG 宏 vs 普通宏**：
> - `SHADER_PARAMETER_RDG_TEXTURE_*` → 字段类型是 `FRDGTextureRef` / `FRDGTextureSRVRef` / `FRDGTextureUAVRef`
> - `SHADER_PARAMETER_TEXTURE` → 字段类型是 `FRHITexture*`（裸 RHI 资源）
>
> **在 RDG Pass 内一律用 RDG 版本**，让 RDG 自动推导屏障。

---

## 5. 内置 Uniform Buffer

引擎提供了若干"开箱即用"的全局 UB，直接在参数结构体里引用即可。

| C++ 类型 | HLSL 名 | 提供什么 |
|----------|---------|---------|
| `FViewUniformShaderParameters` | `View` | 相机与视图：矩阵、屏幕尺寸、时间、曝光、雾、TAA 抖动 |
| `FPrimitiveUniformShaderParameters` | `Primitive` | 图元：LocalToWorld、包围球、光照贴图参数、自定义数据 |
| 材质参数 | `Material` | 材质的标量/向量/纹理参数（由材质系统填充） |
| `FSceneTextureShaderParameters` | （内联展开） | GBuffer / 深度 / 场景色 / 速度缓冲 |

### 5.1 `View` 常用成员

```hlsl
View.ViewToClip                // float4x4
View.TranslatedWorldToClip     // float4x4（相对相机原点的世界→裁剪）
View.ViewSizeAndInvSize        // float4 (W, H, 1/W, 1/H)
View.WorldCameraOrigin         // float3
View.ScreenPositionScaleBias   // float4
View.RealTime                  // float
View.GameTime                  // float
View.ScreenToTranslatedWorld(...)  // 函数：屏幕坐标 → 世界
```

辅助函数（`Common.ush`）：

```hlsl
float2 UV = SvPositionToBufferUV(SvPosition, View);   // SV_Position → [0,1] UV
float2 SvPos = BufferUVToSvPosition(UV, View);        // 反向
float2 ScreenPos = SvPositionToScreenPosition(SvPosition, View);
```

### 5.2 `Primitive` 常用成员

```hlsl
Primitive.LocalToWorld           // float4x4（LWC 下已做平移处理）
Primitive.WorldToLocal
Primitive.ObjectWorldPositionAndRadius   // float4 (xyz=世界位置, w=半径)
Primitive.ObjectBounds                    // float3
```

### 5.3 场景纹理

```cpp
#include "SceneTextureParameters.h"

BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
    SHADER_PARAMETER_STRUCT_INCLUDE(FSceneTextureShaderParameters, SceneTextures)
END_SHADER_PARAMETER_STRUCT()

// 填充
PassParameters->SceneTextures = FSceneTextureShaderParameters::Create(GraphBuilder, View, ...);
```

HLSL 侧：

```hlsl
#include "/Engine/Private/SceneTexturesCommon.ush"

float Depth = SceneTextures.SceneDepthTexture.SampleLevel(...);
float4 GBuffer0 = SceneTextures.GBufferATexture.SampleLevel(...);
```

> 具体成员名随版本演进（UE 5.x 中 `SceneTextures` 曾从全局 UB 改为内联结构体），以引擎头文件 `SceneTextureParameters.h` 为准。

---

## 6. 绑定流程详解

### 6.1 参数结构体到 Uniform Buffer

```
FParameters（C++ POD，字段已填值）
    │
    │  FShaderParametersMetadata（宏生成的反射信息）
    │  遍历每个成员：{ 名字, 类型, 偏移, 大小, 绑定类型 }
    ▼
┌──────────────────────────────────────────┐
│ 标量/向量/矩阵 → memcpy 进 UB 的 [Offset] │
│ 纹理/Sampler  → 记录到资源绑定数组         │
│ UAV/SRV       → 记录到资源绑定数组         │
│ 嵌套结构体    → 递归处理                   │
│ STRUCT_REF    → 绑定已存在的 UB             │
└──────────────────────────────────────────┘
    │
    ▼
FRHIShader + UniformBuffer + 资源绑定 → RDG Pass 执行
```

### 6.2 反射元数据

```cpp
// 拿到参数结构体的元数据（调试/校验用）
const FShaderParametersMetadata* Meta = FParameters::FTypeInfo::GetStructMetadata();

const TArray<FShaderParametersMetadata::FMember>& Members = Meta->GetMembers();
for (const auto& M : Members)
    UE_LOG(LogTemp, Log, TEXT("  %s : offset=%u size=%u"),
           M.GetName(), M.GetOffset(), M.GetMemberSize());
```

### 6.3 编译期校验

宏生成的静态断言会检查：

| 检查 | 失败后果 |
|------|---------|
| C++ 类型大小 == HLSL 类型大小 | 编译报错 |
| 结构体总大小符合对齐规则 | 编译报错 |
| 成员名合法（不含特殊字符） | 编译报错 |
| 使用了 double 类型 | 编译报错（提示改用 float 版） |

> 运行时若参数值不对，多半不是"布局错了"（编译期会拦），而是 **HLSL 变量名与 C++ 字段名不一致**（框架无法检查 HLSL 侧）。

---

## 7. 三种参数传递方式对比

| 方式 | 适用 | 开销 | 生命周期 |
|------|------|------|---------|
| **Uniform Buffer（参数结构体）** | 绝大多数场景 | 低（一次 memcpy + 一次绑定） | 由 RDG / `EUniformBufferUsage` 决定 |
| **Push Constants / 直接 `SetShaderValue`** | 少量标量（UE 内部使用） | 最低 | 单次 draw |
| **纹理 / UAV 直接绑定** | 大块数据 | 中（资源绑定） | 由 RDG 推导 |

> **UE 的实践建议**：能用参数结构体就用参数结构体。不要为了"省一次 memcpy"而手写 `SetShaderValue`——可维护性损失远大于性能收益。

---

## 8. 实战：一个完整参数流

### 8.1 声明

```cpp
// ---------- MyPass.cpp ----------
BEGIN_SHADER_PARAMETER_STRUCT(FMyPassParameters, )
    SHADER_PARAMETER_STRUCT_REF(FViewUniformShaderParameters, View)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D,         SceneColor)
    SHADER_PARAMETER_RDG_TEXTURE_SRV(Texture2D,     SceneColorSRV)
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2D<float4>, OutputUAV)
    SHADER_PARAMETER_SAMPLER(SamplerState,          LinearSampler)
    SHADER_PARAMETER(FVector4f,                     Tint)
    SHADER_PARAMETER(float,                         Intensity)
    SHADER_PARAMETER(uint32,                        PassIndex)
END_SHADER_PARAMETER_STRUCT()
```

### 8.2 填充

```cpp
FMyPassParameters* PassParameters = GraphBuilder.AllocParameters<FMyPassParameters>();
PassParameters->View           = View.ViewUniformBuffer;
PassParameters->SceneColor     = SceneColor.Texture;
PassParameters->SceneColorSRV  = GraphBuilder.CreateSRV(SceneColor.Texture);
PassParameters->OutputUAV      = GraphBuilder.CreateUAV(OutputTexture);
PassParameters->LinearSampler  = TStaticSamplerState<SF_Bilinear>::GetRHI();
PassParameters->Tint           = FVector4f(1, 1, 1, 1);
PassParameters->Intensity      = CVarIntensity.GetValueOnRenderThread();
PassParameters->PassIndex      = 0;
```

### 8.3 HLSL

```hlsl
#include "/Engine/Private/Common.ush"

Texture2D     SceneColor;
Texture2D     SceneColorSRV;
SamplerState  LinearSampler;
RWTexture2D<float4> OutputUAV;

float4 Tint;
float  Intensity;
uint   PassIndex;

[numthreads(8, 8, 1)]
void MainCS(uint3 DispatchThreadId : SV_DispatchThreadID)
{
    uint2 Pixel = DispatchThreadId.xy;
    if (any(Pixel >= uint2(View.ViewSizeAndInvSize.xy)))
        return;

    float2 UV = (float2(Pixel) + 0.5f) * View.ViewSizeAndInvSize.zw;
    float3 Color = Texture2DSampleLevel(SceneColorSRV, LinearSampler, UV, 0).rgb;

    Color *= Tint.rgb * Intensity;
    OutputUAV[Pixel] = float4(Color, 1);
}
```

---

## 9. 常见坑清单

| # | 现象 | 根因 | 对策 |
|---|------|------|------|
| 1 | 编译报 "use FVector3f instead" | 用了 LWC 的 double 类型 | 改 `FVector3f` / `FVector4f` / `FMatrix44f` |
| 2 | 结构体大小不匹配静态断言失败 | `float3` 后成员对齐 | 改用 `FVector4f`，或补 `float` 对齐 |
| 3 | shader 里变量一直是 0 | HLSL 名 ≠ C++ 字段名 | 逐字核对 |
| 4 | 链接报找不到 UB 符号 | 未 `IMPLEMENT_GLOBAL_SHADER_PARAMETER_STRUCT` | 在某个 `.cpp` 里补上 |
| 5 | UAV 读写结果不对 | 用普通宏而非 RDG 宏 → 缺屏障 | 改用 `SHADER_PARAMETER_RDG_*` |
| 6 | 纹理采样全黑 | HLSL 未用 `Texture2DSample` 或 sampler 未绑 | 检查 sampler 字段 |
| 7 | 全屏 Pass 无输出 | 缺少 `RENDER_TARGET_BINDING_SLOTS()` | 补上并填 `RenderTargets[0]` |
| 8 | 参数结构体改了但没生效 | DDC / 热重载未触发 | `recompileshaders changed` |
| 9 | 隐式类型转换警告 | `FLinearColor` 直接塞进结构体 | 用 `FVector4f` |
| 10 | 嵌套结构体成员访问不到 | 用了 `SHADER_PARAMETER_STRUCT` 但 HLSL 写法不对 | 嵌套用 `Name.Field`，内联用 `Field` |

---

## 10. 小结

| 主题 | 要点 |
|------|------|
| 三段式 | 声明（宏生成元数据）→ 填充（赋值 POD）→ 绑定（RDG 自动） |
| 标量/向量 | `SHADER_PARAMETER`，**必须用 float 版类型**（LWC） |
| 资源 | `TEXTURE` / `SRV` / `UAV` / `SAMPLER`；RDG Pass 内用 `RDG_*` 版本 |
| 渲染目标 | `RENDER_TARGET_BINDING_SLOTS()` 是全屏 Pass 必备 |
| 结构体 | `STRUCT`（嵌套）/ `STRUCT_INCLUDE`（内联）/ `STRUCT_REF`（全局 UB） |
| 全局 UB | `BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT` + `IMPLEMENT_..._STRUCT(Name, "HLSLName")` |
| 内置 UB | `View` / `Primitive` / `Material` / `SceneTextures` |
| 校验 | 类型与大小靠编译期静态断言；名字一致性靠人（或工具） |

---

*上一篇：[02-Shader 类型体系与 C++ 绑定](02-Shader类型体系与C++绑定.md) | 下一篇：[04-实战：自定义 Global Shader](04-实战-自定义GlobalShader.md)*
