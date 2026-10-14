# Arm《Advanced graphic techniques - Getting started》中文重点笔记

> 原文：Arm Limited，文档号 102224_0100_00，2020-09-21 首版（Non-Confidential）
> 主题：Unity 中的高级图形技术入门——自定义 Shader、Early-Z、切线空间→世界空间法线转换工具
> 面向：移动端（Arm Mali GPU）图形开发，前提是会 Unity 与基础编程

---

## 1. 总览（Overview）

**学完后应掌握的三件事：**

1. 如何实现顶点 / 片元 Shader（vertex & fragment shaders）
2. Early-Z 是什么、何时会失效
3. 如何使用「切线空间 → 世界空间法线转换工具」

**背景知识：**

- Unity 5+ 内置 **PBS（Physically Based Shading，基于物理的着色）**，集成在标准 Shader（Standard Shader）中；新建材质默认使用标准 Shader。
- 内置 Shader 源码可在 Unity 下载归档中获取（120+ 个 Shader），是学习写法的第一手材料。
- 标准 Shader 做不到的 effects 举例：**基于局部立方体贴图（local cubemap）的反射**——必须自写 Shader。

### 1.1 Unity 自定义 Shader 的三条路线（易混淆，务必分清）

| 方式 | 适用管线 | 特点 |
|------|---------|------|
| **Shader Graph** | 仅 URP / HDRP | 可视化节点编辑，不写代码 |
| **Surface Shader** | 仅内置（标准）管线 | Unity 替你处理光照模型，代码紧凑；适合受光照/阴影影响的材质 |
| **Vertex / Fragment Shader** | 通用 | **最灵活**，但一切自己实现；本文档主线 |

> ⚠️ Unity 的 ShaderLab 不只是顶点/片元 Shader 的容器，但顶点/片元阶段才是图形管线中**真正可编程、做着色计算**的部分。

---

## 2. 自定义 Shader（Custom Shaders）

### 2.1 Shader 结构

最小完整示例（Cg，写在 `CGPROGRAM ... ENDCG` 之间；Unity 也支持 HLSL）：

```cg
Shader "Custom/ctTextured"            // 路径/名字：决定材质下拉菜单中的分类
{
    Properties                        // Inspector 可见、可交互调节的参数
    {
        _AmbientColor ("Ambient Color", Color) = (0.2,0.2,0.2,1.0)
        _MainTex ("Base (RGB)", 2D) = "white" {}
    }
    SubShader                         // 一个 Shader 可含多个 SubShader
    {
        Pass                          // 每个 Pass 让几何体被渲染一次
        {
            CGPROGRAM
            #pragma target 3.0
            #pragma glsl
            #pragma vertex vert       // 指定顶点入口函数
            #pragma fragment frag     // 指定片元入口函数
            #include "UnityCG.cginc"

            uniform float4 _AmbientColor;
            uniform sampler2D _MainTex;

            struct vertexInput {                      // 顶点输入
                float4 vertex   : POSITION;
                float4 texCoord : TEXCOORD0;
            };
            struct vertexOutput {                     // 顶点输出 → 片元输入
                float4 pos : SV_POSITION;
                float4 tex : TEXCOORD0;
            };

            vertexOutput vert(vertexInput input) {
                vertexOutput output;
                output.tex = input.texCoord;
                output.pos = mul(UNITY_MATRIX_MVP, input.vertex);
                return output;
            }
            float4 frag(vertexOutput input) : COLOR {
                float4 texColor = tex2D(_MainTex, input.tex);
                return _AmbientColor + texColor;
            }
            ENDCG
        }
    }
    Fallback "Diffuse"                // 所有 SubShader 都不可用时回退到内置 Diffuse
}
```

**结构要点（按选择顺序记忆）：**

```
Shader "路径/名"
  └─ Properties{}     → Inspector 参数
  └─ SubShader (可多个) → Unity 选「第一个能在当前显卡上运行」的
       └─ Pass (可多个)  → 每个 Pass 渲染几何一次
  └─ Fallback "X"      → 全部 SubShader 失败时的兜底
```

- **多 SubShader 的意义**：硬件/API 不断演进。可为主力平台（如 Mali Midgard + OpenGL ES 3.0）写一个 SubShader，再为老设备（ES 2.0）写一个替代 SubShader。
- **多 Pass 的意义**：老硬件上一个效果可能要分多次渲染才能实现（新硬件一次 Pass 就够）。

### 2.2 编译指令（Compilation Directives）

以 `#pragma` 形式给出。**最少必须包含** `#pragma vertex <name>` 与 `#pragma fragment <name>`。

| 指令 | 作用 | 典型报错/场景 |
|------|------|--------------|
| `#pragma target 3.0` | 编译到更高 Shader Model（默认 2.0） | SM2.0 算术指令上限 64，超了报 `Arithmetic instruction limit of 64 exceeded` → 升 3.0 |
| `#pragma glsl` | 将 Cg/HLSL 转为 GLSL | varying 过多报 `Too many interpolators used`；VS 中用 `tex2D` 报 `not supported in this profile` |
| `#pragma only_renderers ...` | 限定编译目标 API | 移动端只保留 `vulkan gles gles3`，**但编辑器还要 `opengl d3d11`** |

```cg
// 移动端发布 + 编辑器可用的推荐写法：
#pragma only_renderers vulkan gles gles3 opengl d3d11
```

> ⚠️ **易错**：只写 `vulkan gles gles3` 会让 Unity 编辑器（桌面平台）下 Shader 失效，必须把 `opengl`、`d3d11` 一起加上。

### 2.3 Includes

- Unity 提供的 include 位于 `C:\Program Files\Unity\Hub\Editor\<version>\Editor\Data\CGIncludes`。
- **`UnityCG.cginc`**：常用辅助函数与宏，需手动 `#include`。
- **`UnityShaderVariables.cginc`**：**自动包含、无需声明**，提供内置变量。

**性能要点**：矩阵乘法这类「每顶点都一样」的计算，适合在 CPU 做一次再传给 GPU，所以 Unity 预置了组合矩阵。

**常用内置 uniform（重点记忆）：**

| 内置 Uniform | 含义 |
|-------------|------|
| `UNITY_MATRIX_MVP` | 模型 × 视图 × 投影（最常用） |
| `UNITY_MATRIX_MV` | 模型 × 视图 |
| `UNITY_MATRIX_VP` | 视图 × 投影 |
| `UNITY_MATRIX_V` / `UNITY_MATRIX_P` | 视图 / 投影 |
| `_Object2World` | 当前模型矩阵 |
| `_World2Object` | 世界矩阵的逆（用于法线变换） |
| `UNITY_MATRIX_IT_MV` | 模型×视图矩阵的逆转置 |
| `_WorldSpaceCameraPos` | 世界空间相机位置 |
| `_ProjectionParams` | 近/远平面及 1/farPlane |
| `_Time` | 时间向量 (t/20, t, t*2, t*3) |

### 2.4 OpenGL ES 3.0 与 Vulkan 图形管线

> OpenGL ES 与 Vulkan 是两套 API，但**图形管线本质相同**，顶点/片元 Shader 位置一致。

```
┌────────┐  ┌──────────┐  ┌────────┐  ┌────────┐  ┌────────────┐  ┌───────┐
│ Primitives│→│ Vertex  │→│Primitive│→│Rasterizer│→│ Per-fragment│→│ Frame │
│ (输入图元) │  │ Shader  │  │Assembly│  │(光栅化) │  │ operations  │  │Buffer │
└────────┘  └────┬───┘  └────────┘  └───┬────┘  └────────────┘  └───────┘
                 │  ▲                   │
                 │  └──Transform Feedback┘      ▼
                 │                        ┌──────────┐
                 └────────────────────────│ Fragment │
                                          │ Shader   │
                                          └──────────┘
```

各阶段职责：

1. **图元输入**：顶点、点、线、多边形。
2. **顶点 Shader**：可编程，对顶点做变换与光照。
3. **图元装配（Primitive Assembly）**：顶点组装成图元，裁剪到裁剪体，送光栅化器。
4. **光栅化（Rasterizer）**：图元 → 二维片元集合；顶点输出值对每个片元做**插值（interpolation）**。
5. **Transform Feedback**：把顶点 Shader 结果写回缓冲再送回顶点 Shader。⚠️ Unity 不暴露此功能，但内部用它优化**角色蒙皮（skinning）**。
6. **片元 Shader**：可编程，处理每个片元。
7. **逐片元操作（Per-fragment operations）**：依次经过
   - Pixel ownership test（像素归属测试）
   - Scissor test（裁剪测试）
   - Stencil（模板测试）
   - Depth test（深度测试）
   - Blending（混合）
   - Dithering（抖动）
   
   结果只有两种：片元颜色/深度/模板写入帧缓冲，**或**被丢弃。

### 2.5 顶点 Shader（Vertex Shader）

- 对**每个顶点**执行一次：局部空间 3D 坐标 → 屏幕空间 2D 投影坐标 + Z-buffer 深度值。
- **必须返回变换后位置**，否则报错 `function does not return a value: vert`。

```cg
output.pos = mul(UNITY_MATRIX_MVP, input.vertex);   // 位置：MVP 矩阵
```

**法线变换（高频考点/易混淆）：**

- 非均匀缩放后法线不再垂直于三角形 → 法线必须乘**变换矩阵的逆转置**。
- 「转置」操作 = **交换乘法顺序**（向量放左边左乘）：

```cg
// _World2Object 是 local→world 的逆；左乘即实现逆转置变换
float4 normalWorld = mul(float4(input.normal, 0.0), _World2Object);
```

> ⚠️ **第四分量规则**：方向向量（法线/切线）w = **0**（屏蔽平移）；位置坐标 w = **1**。
> ⚠️ 若网格法线已是世界空间，可跳过此步省 VS 计算——**但前提是该网格不会被任何 Unity 内置 Shader 渲染**（内置 Shader 期望法线在物体空间）。

**顶点 Shader 还能做什么：**

- **顶点位移贴图（Displacement Mapping）**：用纹理（如地形高度图）形变网格。
  - 需要 `#pragma target 3.0`（SM3.0 保证 VS 至少可访问 4 个纹理单元）；
  - 强制 OpenGL 渲染时还需 `#pragma glsl`，否则报 `function "tex2D" not supported in this profile`。
- **程序化动画**：用 `_Time` 让顶点坐标随时间变化。
- **网格蒙皮（Mesh Skinning）**：按骨骼动画顶点（Unity 角色动画的基础）。

### 2.6 顶点 Shader 输入

用结构体 + **语义（Semantic）** 声明输入属性：

```cg
struct vertexInput
{
    float4 vertex    : POSITION;    // 顶点位置
    float4 tangent   : TANGENT;     // 切线
    float3 normal    : NORMAL;      // 法线（物体空间）
    float4 texcoord  : TEXCOORD0;   // 第一套 UV
    float4 texcoord1 : TEXCOORD1;   // 第二套 UV（常为 lightmap UV）
    fixed4 color     : COLOR;       // 顶点色
};
```

- **语义** = 附加在输入/输出上的字符串，标明参数用途；**跨阶段传递的变量都必须带语义**。
- 拼错语义会报错：`unknown semantics "TANGENTIAL" specified for "tangent2"`。
- **性能原则：只声明需要的属性。**
- `UnityCG.cginc` 提供预定义输入结构：`appdata_base` / `appdata_tan` / `appdata_full`（上例即等价于 `appdata_full`）——include 后直接用，不必自己声明。

### 2.7 顶点 Shader 输出与 Varying

输出结构**必须包含** `SV_POSITION` 语义标记的变换后坐标，其余可自由添加：

```cg
struct vertexOutput
{
    float4 pos       : SV_POSITION;
    float4 tex       : TEXCOORD0;
    float4 texSpecular : TEXCOORD1;
    float3 vertexInWorld : TEXCOORD2;
    ...
};
```

**插值器（Interpolator）规则（重要限制）：**

- `TEXCOORD0`/`TEXCOORD1` 习惯上留给 UV / lightmap UV，但 `TEXCOORD0~TEXCOORD7` 可传任意数据。
- **每个插值器最多 4 个 float**。传 `float4x4` 矩阵会**连续占用 4 个**插值器（如 `TEXCOORD2`~`TEXCOORD5`）。

**插值原理（核心概念）：**

- 顶点 → 片元传递的一切 varying，默认由**硬件线性插值器**在光栅化阶段做**线性插值**。
- 三角形三顶点 V1/V2/V3，像素值用**重心坐标（barycentric coordinates）λ1、λ2、λ3** 加权插值。

```
        V1(红)
       /      \
      /   P    \      P 点颜色 = λ1·红 + λ2·绿 + λ3·蓝
     /  λ1λ2λ3  \
    V2(绿)──────V3(蓝)
```

**性能推论（关键优化直觉）：**

- 线性插值的值，**可以在顶点 Shader 算好再传**，不必在片元 Shader 里逐像素算。
- 例：想要「颜色 = 到某点 C 的距离的函数」，就在 VS 里算「顶点到 C 的平方距离」作为 varying 传出，硬件自动帮你插值到每个像素。
- 顶点数 ≪ 片元数 → **把计算从 FS 移到 VS 是显著的性能收益**。

> ⚠️ **移动端反面警告**：varying 越多，顶点读取与 FS varying 读取的**带宽消耗**越大。移动端带宽宝贵，要在数据量与画质间取平衡。

### 2.8 片元 Shader（Fragment Shader）

- 位于光栅化**之后**；图元覆盖的每个像素样本生成一个片元，**逐片元执行**。
- **片元远多于顶点** → 严格控制 FS 内的运算量。
- FS 可访问：3D 屏幕空间的片元坐标 + 所有从 VS 插值而来的值。
- 入口声明 `: COLOR` 表明输出片元颜色：

```cg
float4 frag(vertexOutput input) : COLOR {
    float4 texColor = tex2D(_MainTex, input.tex);   // 纹理采样
    return _AmbientColor + texColor;                // 叠加环境色
}
```

**一句话记忆：VS 负责「在哪」，FS 负责「长什么样」。**

### 2.9 向 Shader 传数据

`Pass` 块中声明的 **uniform** 对 VS 和 FS 都可见；uniform 类似**全局常量**，Shader 内不可修改。

**两种传数据方式：**

| 方式 | 适用场景 | 支持的类型 / API |
|------|---------|-----------------|
| `Properties` 块 | 开发期交互调参，运行时实时看效果 | Float、Color、Texture 2D、Cubemap、Rectangle、Vector |
| 脚本程序化传值 | 依赖前序计算结果、或需在特定时机传值 | `Material.SetColor / SetFloat / SetInt / SetMatrix / SetVector / SetTexture` |

**实战示例（阴影纹理投影）**：主相机渲染前，用阴影相机 `shwCam` 把阴影渲成纹理，再把矩阵和纹理传给各材质：

```csharp
public void OnWillRenderObject()           // 对象渲染前回调
{
    CreateShadowsTexture();
    // 传矩阵
    for (int i = 0; i < shwMats.Count; i++)
    {
        shwMats[i].SetMatrix("_ShwCamProjMat", shwCam.projectionMatrix);
        shwMats[i].SetMatrix("_ShwCamViewMat",  shwCam.transform.worldToLocalMatrix);
    }
    shwCam.Render();                        // 渲染阴影纹理
    // 传纹理
    for (int i = 0; i < shwMats.Count; i++)
        shwMats[i].SetTexture("_ShadowsTex", m_ShadowsTexture);
}
```

传给 Shader 的三样数据 ↔ Shader 内对应 uniform：

- 阴影相机投影矩阵 → `_ShwCamProjMat`
- 世界→局部变换矩阵（`worldToLocalMatrix`）→ `_ShwCamViewMat`
- 阴影纹理 → `m_ShadowsTexture`

### 2.10 在 Unity 中调试 Shader

**Shader 无法像普通代码那样断点调试** → 用**片元颜色输出可视化中间值**，靠「看图说话」。

**方法**：把想调试的向量归一化后直接作为颜色输出：

```cg
return float4(normalize(localCorrReflDirWS), 1.0);   // 反射向量 → 颜色
```

**颜色解读（以局部 cubemap 反射调试为例）：**

- 偏红 = 反射向量 X 分量强（反射来自带窗的墙）；
- 偏蓝 = Z 分量强（反射来自右墙）；
- 黑色 = 主要朝 **-Z** 方向——**负分量被钳制到 0**，不可见。

**调试技巧：**

1. 多分量混在一起难以解读时，**只看单分量**：

```cg
float3 n = normalize(localCorrReflDirWS);
return float4(0, n.y, 0, 1);    // 只看 Y：只显示来自天花板的反射，墙面全黑
```

2. > ⚠️ **铁律：被调试的量必须先缩放到 [0,1]**——超出范围会被自动钳制（<0 → 0，>1 → 1），信息丢失。

---

## 3. Early-Z

**定义**：Mali GPU 内置的**提前深度测试**机制——在**运行片元 Shader 之前**先做深度检查，剔除被遮挡的片元，避免 GPU 为最终不可见的片元浪费着色算力（减少 overdraw 代价）。

**失控点**：Early-Z 是否生效**很难在 Unity 中直接控制**（取决于引擎 + 编译器生成的代码），但可从代码特征判断。

**以下 4 类情况会导致 Early-Z 失效或结果错误（核心记忆表）：**

| 情况 | 为什么 Early-Z 不能用 |
|------|---------------------|
| **Shader 有副作用（side effects）** | 线程执行中修改全局状态（如写 SSBO / image 的共享读写缓冲），重复执行结果不同。例：Shader 内做计数器统计性能 |
| **FS 中调用 `discard()`** | FS 可能丢弃片元，但 Early-Z **已提前写过深度值且无法撤销** |
| **启用 Alpha-to-coverage** | FS 计算的 alpha 之后才用于覆盖率。例：树叶面片的透明区会错误遮挡背后场景 |
| **FS 写 `gl_FragDepth`** | 深度值不再来自顶点 Shader，Early-Z 无深度可用 |

> ✅ **不算副作用**的三类操作：**只读**内存访问；写**只写（write-only）**缓冲；**纯局部**内存访问。

---

## 4. 切线空间 → 世界空间法线转换工具

### 4.0 动机（为什么做这个工具）

- Ice Cave demo 性能分析发现**算术管线是瓶颈**。
- 切线空间法线贴图：适合**动画/动态**物体，但采样后需额外计算把法线转到正确朝向。
- 场景中**大部分几何是静态的** → 离线把法线贴图转成**世界空间**，Shader 采样出来**直接可用**，省去运行时 TBN 变换。
- **前提**：光照在自定义 Shader 中计算（Unity 标准 Shader 用的是切线空间法线贴图，此优化不适用于它）。
- 工具 = **C# 脚本（加编辑器菜单）+ 转换 Shader**，**离线运行，零运行时开销**。

### 4.1 C# 脚本 WorldSpaceNormalsCreator

**用法要点：**

- 脚本必须放在 **`Assets/Editor`** 目录（不存在则自建），才能在 GameObject 菜单加选项。
- 类继承 **`ScriptableWizard`**（编辑器向导基类）：
  - `OnWizardUpdate()`：设置 `helpString` 帮助文本；用 `isValid` 控制 Create 按钮可用性（此处检查 `_currentObj != null`）。
  - 向导窗口的字段 = 类的 **public 成员**（此类只有 `_currentObj` 一个字段）。
  - 点 Create → 调用 `OnWizardCreate()` 执行转换。
- 菜单注册：

```csharp
[MenuItem("GameObject/World Space Normals Creator")]
static void CreateWorldSpaceNormals()
{
    ScriptableWizard.DisplayWizard("Create World Space Normal",
        typeof(WorldSpaceNormalsCreator), "Create");
}
```

**`OnWizardCreate()` 转换流程（核心逻辑）：**

```
1. QualitySettings.antiAliasing = 4
2. Shader.Find("Custom/WorldSpaceNormalCreator") 取转换 Shader
3. 建临时正交相机（orthographic, near=0, far=10, size=1）
4. 目标物体改到未使用层 layer 30 → 单独渲染它
5. SetReplacementShader(wns, null)  +  useOcclusionCulling = false
6. 相机转向物体，避免视锥剔除把它剔掉
7. 遍历物体每个材质：
   a. 取 _BumpMap 纹理（无则报错跳过）
   b. Shader.SetGlobalTexture("_BumpMapGlobal", t) 设为全局纹理
   c. 建同尺寸 RenderTexture，背景色 (0.5, 0.5, 0.5)，cullingMask=0x40000000
   d. camera.Render() 渲染 → ReadPixels 读回
   e. EncodeToPNG → 存到 Assets/Textures/GeneratedWorldSpaceNormals/
8. 恢复物体原 layer，销毁临时相机
```

**细节解读：**

- **替换 Shader（Replacement Shader）**：相机渲染时强制用转换 Shader 替换物体原有 Shader。
- **背景色 (0.5, 0.5, 0.5)**：法线有负方向分量，映射后中灰对应 (0,0,0) 法线。
- **cullingMask = 0x40000000**：二进制掩码的十六进制表示，**第 30 位（layer 30）置 1**，即 `1 << 30`。

### 4.2 转换 Shader WorldSpaceNormalCreator

**核心技巧：不用顶点位置，改用 UV 当位置**——把物体「按 UV 摊平」到 2D 平面渲染（就像贴图展开）：

```cg
// UV 从 [0,1] 映射到裁剪空间 [-1,1]，并翻转 Y；Z 不用，取裁剪面内任意值
output.pos = half4(input.tex.x*2.0 - 1.0, ((1.0-input.tex.y)*2.0 - 1.0), 0.0, 1.0);
output.tc  = input.tex;
```

**顶点 Shader：构建世界空间 TBN 基**

```cg
output.normalInWorld  = normalize(mul(half4(input.normal, 0.0), _World2Object).xyz);   // 逆转置（左乘）
output.tangentWorld   = normalize(mul(_Object2World, half4(input.tangent.xyz, 0.0)).xyz);
output.bitangentWorld = normalize(cross(output.normalInWorld, output.tangentWorld)
                                  * input.tangent.w);   // w 分量决定副切线手性
```

> ⚠️ `tangent.w` 存**手性（±1）**，`cross(N, T)` 乘上它才得到正确方向的副切线。

**片元 Shader：三步完成转换**

```cg
half3 bumpNormal = UnpackNormal(tex2D(_BumpMapGlobal, input.tc));   // 1. 采样并解包切线空间法线
half3x3 local2WorldTranspose = half3x3(input.tangentWorld,
                                       input.bitangentWorld,
                                       input.normalInWorld);
normalInWorld = normalize(mul(bumpNormal, local2WorldTranspose));   // 2. 切线空间 → 世界空间（左乘 TBN 转置）
normalInWorld = normalInWorld * 0.5 + 0.5;                          // 3. 从 [-1,1] 缩放到 [0,1] 存入纹理
return half4(normalInWorld, 1.0);
```

> 效果：输入（图11）蓝紫色调的切线空间法线图 → 输出（图12）彩色方向各异的世界空间法线图。

---

## 5. 术语与易混淆点速查

| 术语/概念 | 要点 | 易混淆 |
|-----------|------|--------|
| **PBS** | Unity 5+ 标准 Shader 的基于物理着色，不同光照下外观一致 | ≠ 必须手写；默认材质即用它 |
| **Surface Shader vs V/F Shader** | 前者 Unity 代管光照、仅内置管线；后者全手动、最灵活 | Shader Graph 仅 URP/HDRP |
| **SubShader vs Pass vs Fallback** | SubShader 按硬件能力择优；Pass = 几何渲染次数；Fallback = 全失败兜底 | 多 SubShader ≠ 多 Pass |
| **Semantic（语义）** | 标记参数用途的字符串，跨阶段传参必须带 | `POSITION`（输入）vs `SV_POSITION`（VS 输出） |
| **插值器上限** | 每个 `TEXCOORDn` 最多 4 float；`float4x4` 占 4 个连续插值器 | varying 太多 → 移动端带宽爆炸 |
| **向量 vs 坐标变换** | 方向向量 w=0，位置 w=1 | 法线用**逆转置**（左乘 `_World2Object`），位置用 MVP |
| **计算下沉 VS** | 线性插值的量可在 VS 算好传出，省 FS 算力 | 非线性关系（如 normalize）不能这么移 |
| **Shader 调试** | 输出中间向量当颜色；单分量排查；**必须先归一化到 [0,1]** | 负值被钳到 0 → 黑色不代表「零」 |
| **Early-Z** | FS 前的提前深度剔除，省 overdraw | 副作用 / `discard` / Alpha-to-coverage / 写 `gl_FragDepth` 会禁用它 |
| **切线空间 vs 世界空间法线图** | 前者通用（动画物体）但运行时需 TBN 变换；后者静态物体采样即用，省 ALU | 世界空间法线图**不能用于会动/会变形的物体** |
| **替换 Shader** | `Camera.SetReplacementShader` 用指定 Shader 渲染物体，常用于烘焙工具 | 配合独立 layer + cullingMask 隔离渲染 |
| **tangent.w** | 副切线手性 ±1 | `cross(N,T)` 必须乘 `tangent.w` |

---

## 6. 小结

| # | 核心结论 |
|---|---------|
| 1 | Unity 自定义 Shader 三路线：Shader Graph（URP/HDRP）、Surface Shader（内置管线）、V/F Shader（最灵活） |
| 2 | Shader 骨架：`Shader名 → Properties → SubShader(择优) → Pass(次数) → Fallback(兜底)` |
| 3 | 默认 Shader Model 2.0 指令上限 64，大型 Shader 必须 `#pragma target 3.0`；移动端限定 `only_renderers` 时别忘加编辑器的 `opengl d3d11` |
| 4 | 法线变换 = 逆转置矩阵 = 左乘 `_World2Object`；方向向量 w=0、位置 w=1 |
| 5 | varying 默认硬件线性插值（重心坐标）；**能放 VS 算的别放 FS**，但移动端要控制 varying 数量省带宽 |
| 6 | Shader 调试 = 把中间量归一化后当颜色输出；单分量排查；超 [0,1] 会被钳制 |
| 7 | Early-Z 在 FS 前剔除被遮挡片元；`discard`、Shader 副作用、Alpha-to-coverage、写 `gl_FragDepth` 会让它失效 |
| 8 | 静态几何可用「切线空间→世界空间法线」离线烘焙工具省运行时 TBN 计算；工具核心 = UV 当位置的正交相机替换 Shader 渲染 |
