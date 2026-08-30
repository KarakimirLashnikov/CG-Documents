# SPIR-V 模块结构与反射原理

> 系列第 **1** 篇。反射库的"魔法"其实只有一句话：**SPIR-V 是一份自带元数据的自描述二进制**。本文拆开这份二进制，讲清楚反射库到底在读什么、怎么读、以及哪些东西它读不到。

---

## 1. SPIR-V 是什么，不是什么

### 1.1 定位

SPIR-V（Standard Portable Intermediate Representation - V）是 Khronos 定义的**二进制中间表示**，同时服务于 Vulkan、OpenCL、SYCL。它不是：

- ❌ 不是"SPIR 的 V 版"——SPIR 是早期基于 LLVM IR bitcode 的另一套格式，早已被 SPIR-V 取代
- ❌ 不是汇编——它没有寄存器名、没有指令编码表，是**语义级 IR**
- ❌ 不是文本——需要 `spirv-dis` 反汇编后人类才能读

它的设计目标是：**让驱动厂商不用写编译器前端**。前端（glslang / DXC / clang）负责把源码翻译成 SPIR-V，驱动只做 SPIR-V → 本机 ISA 的后端工作。

### 1.2 三条核心性质（决定了反射能做到什么）

| 性质 | 含义 | 对反射的影响 |
|------|------|-------------|
| **SSA + 显式 CFG** | 每个值只赋值一次，控制流用 `OpBranch`/`OpLabel` 显式表达 | 没有"变量"概念，只有 ID；反射靠 `OpVariable`（带 storage class 的指针）识别资源 |
| **Type-centric，ID 引用** | 每条产生结果的指令分配一个 `<id>`，形成 DAG | 反射需要先建 ID → 类型/指令的映射，再回溯 |
| **Decoration 外置** | 绑定号、offset、location 等全部用 `OpDecorate` 挂在 ID 上 | ⭐ 反射可以**只读 Decoration 段**就拿到全部绑定信息，不必理解函数体 |

第三条是 SPIRV-Reflect 这种"轻量解析器"能存在的原因：它**不需要理解任何一条函数体指令**。

---

## 2. 二进制布局

### 2.1 文件头（固定 5 个字 = 20 字节）

```
word 0    ┌──────────────────────────────────────┐
          │ Magic Number = 0x07230203            │  小端存储
word 1    ├──────────────────────────────────────┤
          │ Version  (major<<16)|(minor<<8)|rev  │  0x00010600 = 1.6
word 2    ├──────────────────────────────────────┤
          │ Generator Magic Number               │  工具 ID<<16 | 版本
word 3    ├──────────────────────────────────────┤
          │ Bound  = 最大 ID + 1                 │  ← 反射库据此预分配 ID 表
word 4    ├──────────────────────────────────────┤
          │ Schema = 0 (保留)                    │
          └──────────────────────────────────────┘
          ┌──────────────────────────────────────┐
word 5... │ 指令流 (Instruction Stream)           │
          └──────────────────────────────────────┘
```

**Generator Magic Number** 高 16 位是工具 ID（Khronos 统一分配），例如：

| 值 | 工具 |
|----|------|
| 6 | Khronos LLVM/SPIR-V Translator |
| 7 | Khronos SPIR-V Tools Assembler |
| 8 | Khronos glslang Reference Front End |
| 4 | NVIDIA |
| 10 | AMD |

> SPIRV-Reflect 在 `spirv_reflect.h` 中定义了 `SpvReflectGenerator` 枚举来识别它，你可以用 `module.generator` 判断"这个 spv 是哪个工具链产出的"，在排查兼容性问题时很有用（例如 glslang 与 DXC 的矩阵布局默认不同）。

### 2.2 指令编码

每条指令的第一个字同时编码了长度和操作码：

```
 31                              16 15                               0
┌──────────────────────────────────┬──────────────────────────────────┐
│          WordCount (16 bit)      │           Opcode (16 bit)        │
└──────────────────────────────────┴──────────────────────────────────┘
┌─────────────────────────────────────────────────────────────────────┐
│ Operand 1 (可选, 可能是 <id> / 字面量 / 字符串)                       │
├─────────────────────────────────────────────────────────────────────┤
│ Operand 2 ...                                                        │
└─────────────────────────────────────────────────────────────────────┘
```

**字符串操作数**以 `\0` 结尾、按 4 字节对齐填充，可能占据多个字——这是手写解析器最容易踩的坑（必须用 `WordCount` 跳转，不能按操作数个数跳转）。

### 2.3 模块逻辑分段（Logical Layout）

规范 §2.4 规定了模块的**十一个逻辑段**，顺序是强制的：

```
┌─────────────────────────────────────────────────────────────┐
│  ① OpCapability ............ 能力声明（Shader, Float16, ...） │
│  ② OpExtension ............. 扩展（SPV_KHR_...）             │
│  ③ OpExtInstImport ......... 扩展指令集（GLSL.std.450 等）    │
├─────────────────────────────────────────────────────────────┤
│  ④ OpMemoryModel ........... 单一必选：寻址+内存模型          │
│                              Vulkan 固定为 Logical GLSL450    │
├─────────────────────────────────────────────────────────────┤
│  ⑤ OpEntryPoint ............ 入口点：执行模型 + 名字 + 接口表  │
│  ⑥ OpExecutionMode ......... 执行模式（LocalSize, Origin...） │
├─────────────────────────────────────────────────────────────┤
│  ⑦ Debug#1：OpString / OpSource / OpSourceContinued          │
│  ⑧ Debug#2：OpName / OpMemberName ....... ★ 名字反射的来源     │
├─────────────────────────────────────────────────────────────┤
│  ⑨ Annotations：OpDecorate / OpMemberDecorate / OpDecorateId │
│                              ......... ★ 绑定/location 来源    │
├─────────────────────────────────────────────────────────────┤
│  ⑩ 类型/常量/全局变量：OpType* / OpConstant* / OpVariable     │
├─────────────────────────────────────────────────────────────┤
│  ⑪ 函数定义：OpFunction ... OpFunctionEnd                    │
└─────────────────────────────────────────────────────────────┘
```

> **反射库的工作区间就是 ⑤⑥⑧⑨⑩**，函数体（⑪）里除了少数情况（如特化常量 Op、字节地址缓冲偏移）基本不用碰。

---

## 3. 一个真实模块长什么样

以这段 HLSL（Shader Model 6.x，用 DXC 编译）为例：

```hlsl
cbuffer PerFrame : register(b0, space0) {
    float4x4 viewProj;   // 64 bytes
    float4   cameraPos;  // 16 bytes
};

Texture2D     albedoTex     : register(t0, space0);
SamplerState  linearSampler : register(s0, space0);

struct VSOut { float4 pos : SV_Position; float2 uv : TEXCOORD0; };

float4 PSMain(VSOut i) : SV_Target {
    return albedoTex.Sample(linearSampler, i.uv) * cameraPos;
}
```

`spirv-dis` 输出（节选，省略了函数体与部分类型）：

```asm
; ① 能力
               OpCapability Shader
; ④ 内存模型
               OpMemoryModel Logical GLSL450
; ⑤ 入口点：Fragment 执行模型，名字 "PSMain"，接口 = [输出变量, uv]
               OpEntryPoint Fragment %PSMain "PSMain" %out_var_SV_TARGET %in_var_TEXCOORD0
; ⑥ 执行模式
               OpExecutionMode %PSMain OriginUpperLeft
; ⑦ 源信息
               OpSource HLSL 610
; ⑧ 名字（反射的 name 来源）
               OpName %PerFrame "PerFrame"
               OpMemberName %PerFrame_type 0 "viewProj"
               OpMemberName %PerFrame_type 1 "cameraPos"
               OpName %albedoTex "albedoTex"
               OpName %linearSampler "linearScaler"
               OpName %in_var_TEXCOORD0 "i.uv"
; ⑨ 注解（反射的 binding / offset 来源）
               OpDecorate %out_var_SV_TARGET Location 0
               OpDecorate %in_var_TEXCOORD0 Location 0
               OpDecorate %PerFrame DescriptorSet 0          ; ← set = 0
               OpDecorate %PerFrame Binding 0                ; ← binding = 0
               OpDecorate %PerFrame Block                    ; ← 是 UBO
               OpMemberDecorate %PerFrame_type 0 Offset 0    ; ← viewProj @ 0
               OpMemberDecorate %PerFrame_type 0 RowMajor    ; ← 行主序!
               OpMemberDecorate %PerFrame_type 0 MatrixStride 16
               OpMemberDecorate %PerFrame_type 1 Offset 64   ; ← cameraPos @ 64
               OpDecorate %albedoTex DescriptorSet 0
               OpDecorate %albedoTex Binding 1
               OpDecorate %linearSampler DescriptorSet 0
               OpDecorate %linearSampler Binding 2
; ⑩ 类型与全局变量
      %float = OpTypeFloat 32
    %v4float = OpTypeVector %float 4
 %mat4v4float = OpTypeMatrix %v4float 4                       ; float4x4
%PerFrame_type = OpTypeStruct %mat4v4float %v4float           ; 结构体 = UBO 布局
%_ptr_Uniform_PerFrame_type = OpTypePointer Uniform %PerFrame_type
   %PerFrame = OpVariable %_ptr_Uniform_PerFrame_type Uniform ; ← 存储类 = Uniform
      %image = OpTypeImage %float 2D 2 0 0 1 Unknown          ; 2D, sampled=2(?), 非数组, 非MS
%_ptr_UC_image = OpTypePointer UniformConstant %image
  %albedoTex = OpVariable %_ptr_UC_image UniformConstant
    %sampler = OpTypeSampler
```

**对照这张图，反射库要做的事就一目了然了**：

| 反射输出 | 读自哪里 |
|---------|---------|
| `binding->name = "albedoTex"` | `OpName %albedoTex` |
| `binding->set = 0, binding = 1` | `OpDecorate %albedoTex DescriptorSet/Binding` |
| `binding->descriptor_type = SAMPLED_IMAGE` | `OpTypeImage` + storage class（未与 sampler 合并时） |
| `block.members[1].offset = 64` | `OpMemberDecorate %PerFrame_type 1 Offset 64` |
| `block.members[0].decoration_flags = ROW_MAJOR` | `OpMemberDecorate ... RowMajor` |
| `input->location = 0` | `OpDecorate %in_var_TEXCOORD0 Location 0` |
| `module->shader_stage = FRAGMENT` | `OpEntryPoint Fragment` |

---

## 4. 描述符类型的判定规则

这是反射库最核心的一张映射表。SPIR-V **没有** `OpDecorate DescriptorType`，类型只能靠 **类型构造 + 存储类** 推导：

| 存储类 | 变量类型 | 推导出的 `VkDescriptorType` |
|--------|---------|------------------------------|
| `UniformConstant` | `OpTypeSampler` | `SAMPLER` |
| `UniformConstant` | `OpTypeImage` (Sampled=1, Dim≠Buffer) | `SAMPLED_IMAGE` |
| `UniformConstant` | `OpTypeImage` (Sampled=2, Dim≠Buffer, 非 SubpassData) | `STORAGE_IMAGE` |
| `UniformConstant` | `OpTypeImage` (Dim=SubpassData) | `INPUT_ATTACHMENT` |
| `UniformConstant` | `OpTypeImage` (Sampled=1, Dim=Buffer) | `UNIFORM_TEXEL_BUFFER` |
| `UniformConstant` | `OpTypeImage` (Sampled=2, Dim=Buffer) | `STORAGE_TEXEL_BUFFER` |
| `UniformConstant` | `OpTypeSampledImage` | `COMBINED_IMAGE_SAMPLER` |
| `UniformConstant` | `OpTypeAccelerationStructureKHR` | `ACCELERATION_STRUCTURE_KHR` |
| `Uniform` | `OpTypeStruct` + `Block` | `UNIFORM_BUFFER` |
| `StorageBuffer` | `OpTypeStruct` | `STORAGE_BUFFER` |
| `Uniform` | `OpTypeStruct` + `BufferBlock` | `STORAGE_BUFFER`（SPIR-V <1.3 旧式） |
| `PushConstant` | `OpTypeStruct` | （不是描述符，是 PushConstantRange） |

### 4.1 `OpTypeImage` 的操作数

```
OpTypeImage  <result-id>  <sampled-type>  <dim>  <depth>  <arrayed>  <MS>  <sampled>  <format>
                              │              │       │        │        │       │         │
                              │              │       │        │        │       │         └─ 图像格式
                              │              │       │        │        │       └─ 0=未知/运行时 1=可采样 2=存储
                              │              │       │        │        └─ 0/1 多重采样
                              │              │       │        └─ 0/1 数组纹理
                              │              │       └─ 0=非深度 1=深度 2=未知
                              │              └─ 1D/2D/3D/Cube/Rect/Buffer/SubpassData
                              └─ 采样返回的分量类型
```

SPIRV-Reflect 把这张表拆成了 `binding->image.dim / depth / arrayed / ms / sampled / image_format`。

### 4.2 `COMBINED_IMAGE_SAMPLER` 从哪来

这是最容易被误解的一点。SPIR-V 里**没有"合并采样器"这个类型**，它来自 DXC/glslang 的优化决策：

- `albedoTex.Sample(sampler, uv)` 中 texture 与 sampler 被**成对**使用 → 前端可能把 `OpTypeImage + OpTypeSampler` 折叠成 `OpTypeSampledImage`，共用一个 binding
- DXC 默认**不会**自动合并；需要显式 `[[vk::combinedImageSampler]]` 或在 GLSL 中写 `sampler2D`（GLSL 的 `sampler2D` 天然就是合并的）

> **工程建议**：反射得到 `COMBINED_IMAGE_SAMPLER` 时，你的描述符更新代码必须**同时写入 image view 和 sampler** 到一个 `VkDescriptorImageInfo`。很多"画面全黑"的 bug 源于此。

### 4.3 数组与运行时长度数组（Bindless）

```
Texture2D g_Tex[8];        →  OpTypeArray<OpTypeImage, 8>
                              binding->count = 8, array.dims[0] = 8

Texture2D g_Bindless[];    →  OpTypeRuntimeArray<OpTypeImage>   （HLSL 里用 unsized array）
                              binding->array.dims[0] = 0   ← 长度未知
```

**SPIRV-Reflect 的约定**：运行时长度数组的维度报告为 `0`（或 `count` 为 0）。引擎必须自己决定上界：

```cpp
uint32_t descCount = (b.count > 0) ? b.count : kBindlessCapacity;
bool     runtimeSized = (b.count == 0 || (b.array.dims_count && b.array.dims[0] == 0));
```

对应的 Vulkan 标志：

```cpp
if (runtimeSized) {
    bindingFlags = VK_DESCRIPTOR_BINDING_PARTIALLY_BOUND_BIT
                 | VK_DESCRIPTOR_BINDING_UPDATE_AFTER_BIND_BIT
                 | VK_DESCRIPTOR_BINDING_VARIABLE_DESCRIPTOR_COUNT_BIT;
}
```

---

## 5. 内存布局：Block 的成员偏移

UBO / SSBO 的成员布局信息全部来自 `OpMemberDecorate`：

```
OpMemberDecorate %PerFrame_type 0 Offset 0
OpMemberDecorate %PerFrame_type 0 MatrixStride 16
OpMemberDecorate %PerFrame_type 0 RowMajor
OpMemberDecorate %PerFrame_type 1 Offset 64
```

SPIRV-Reflect 把它解析成：

```c
typedef struct SpvReflectBlockVariable {
  uint32_t                    spirv_id;
  const char*                 name;
  uint32_t                    offset;          // 相对父结构
  uint32_t                    absolute_offset; // 相对块起点（含嵌套）
  uint32_t                    size;            // 标量/向量/矩阵的实际字节数
  uint32_t                    padded_size;     // 含 std140/std430 padding
  SpvReflectDecorationFlags   decoration_flags;
  SpvReflectNumericTraits     numeric;         // 标量宽/符号、向量分量数、矩阵行列/stride
  SpvReflectArrayTraits       array;           // 数组维度与 stride
  uint32_t                    flags;
  uint32_t                    member_count;
  SpvReflectBlockVariable*    members;         // 递归展开嵌套结构
  SpvReflectTypeDescription*  type_description;
} SpvReflectBlockVariable;
```

### 5.1 `std140` vs `std430` vs HLSL 打包

| 布局规则 | 适用 | 关键差异 |
|---------|------|---------|
| **std140** | UBO | vec3 占 16 字节；数组元素 stride 至少 16 字节 |
| **std430** | SSBO / PushConstant | vec3 占 12 字节；数组元素可紧凑 |
| **HLSL 默认（CBV）** | D3D 风格 cbuffer | 不允许跨 16 字节寄存器边界；**矩阵默认列主序** |
| **`--hlsl-layout`/`-fvk-use-dx-layout`** | DXC | 按 HLSL 规则打包，与 D3D12 一致 |

> ⚠️ **跨编译器陷阱**：glslang 默认对 `float4x4` 生成 `ColMajor`，HLSL/DXC 默认 `RowMajor`。同一个 HLSL 源码用两个编译器编出来的 UBO 字节布局可能不同。反射能告诉你**实际**是哪一种（`decoration_flags & SPV_REFLECT_DECORATION_ROW_MAJOR`），所以**永远以反射结果为准，不要假设**。

### 5.2 生成 C++ 结构体时的两条铁律

```cpp
// 1) 用反射的 absolute_offset 生成成员，不要自己算
struct PerFrame {
    glm::mat4 viewProj;   // @ 0   (RowMajor, stride 16)
    glm::vec4 cameraPos;  // @ 64
};

// 2) 用 static_assert 把"反射事实"钉死在编译期
static_assert(offsetof(PerFrame, viewProj)  == 0);
static_assert(offsetof(PerFrame, cameraPos) == 64);
static_assert(sizeof(PerFrame) == 80);
```

第 2 步是**代码生成的核心价值**：shader 一旦改了，C++ 侧要么重新生成、要么**编译不过**——而不是运行时静默错位。

---

## 6. 入口点与接口变量

### 6.1 `OpEntryPoint` 的 interface 列表

```
OpEntryPoint <execution-model> <entry-point-id> "<name>" <interface-id>...
```

这个列表的内容**随 SPIR-V 版本变化**，是反射的一个大坑：

| SPIR-V 版本 | interface 列表内容 | 后果 |
|-------------|-------------------|------|
| **≤ 1.3** | 仅 `Input` / `Output` 存储类的全局变量 | `spvReflectEnumerateEntryPointDescriptorBindings` 可能返回**空** |
| **≥ 1.4** | **所有**被该入口点静态使用的全局变量（含描述符、PushConstant） | 入口点过滤可用 |

> **排查口诀**：如果 `EntryPoint*` 系列 API 返回 0 条，先确认 SPIR-V 版本（`-fspv-target-env=vulkan1.3` 会产出 1.6）。否则退回模块级 `spvReflectEnumerate*`。

### 6.2 接口变量能拿到什么

```c
typedef struct SpvReflectInterfaceVariable {
  uint32_t                      spirv_id;
  const char*                   name;          // 变量名（来自 OpName）
  uint32_t                      location;      // OpDecorate Location
  uint32_t                      component;     // OpDecorate Component
  SpvStorageClass               storage_class; // Input / Output / ...
  const char*                   semantic;      // HLSL 语义（SV_Position / TEXCOORD0）
  SpvReflectDecorationFlags     decoration_flags; // FLAT / NOPERSPECTIVE / PATCH / PER_VERTEX
  SpvBuiltIn                    built_in;      // OpDecorate BuiltIn（非 0 则是内建量）
  SpvReflectNumericTraits       numeric;
  SpvReflectArrayTraits         array;
  uint32_t                      member_count;
  SpvReflectInterfaceVariable*  members;
  SpvReflectFormat              format;        // 数值与 VkFormat 对齐
  SpvReflectTypeDescription*    type_description;
} SpvReflectInterfaceVariable;
```

**用途**：
- 生成 `VkVertexInputAttributeDescription`（顶点着色器的 Input 变量）
- 做 VS 输出 → PS 输入的**匹配校验**（location + component + 类型必须一致）
- 片元着色器输出数量 → 判断 MRT 附件数
- 计算着色器无输入/输出，但有 `LocalSize`（在 `SpvReflectEntryPoint::local_size`）

### 6.3 内建量（BuiltIn）必须跳过

`SV_Position` / `gl_FragCoord` / `gl_VertexIndex` 这类内建量**不占用用户 location**，生成顶点输入布局时必须过滤：

```cpp
for (auto* v : inputs) {
    if (v->built_in != SpvBuiltInMax) continue;   // 内建量，跳过
    if (!(v->decoration_flags & SPV_REFLECT_DECORATION_BUILT_IN)) {
        // 这才是真正的用户顶点属性
    }
}
```

---

## 7. 反射库的内部工作原理

理解了模块结构，就能理解反射库的解析流程。SPIRV-Reflect 的 `Parse()` 大致是**四遍扫描**：

```
┌─ Pass 1：线性扫描 + 词法解析 ──────────────────────────────┐
│  读 header → 校验 magic(0x07230203) → 逐条指令切分          │
│  建立：ID → 指令起始位置的索引表（大小 = header.bound）      │
│  顺手收集：OpCapability / OpExtension / OpEntryPoint        │
│           OpMemoryModel / OpExecutionMode                  │
└──────────────────────────┬─────────────────────────────────┘
                           ▼
┌─ Pass 2：类型与变量收集 ───────────────────────────────────┐
│  遍历 ⑩ 段：                                                │
│   OpType*       → 建 SpvReflectTypeDescription（递归展开）  │
│   OpVariable    → 按 storage_class 分流：                  │
│                    UniformConstant → 描述符候选             │
│                    Uniform/StorageBuffer → Block 候选       │
│                    PushConstant → PushConstant 候选         │
│                    Input/Output → 接口变量候选              │
│   OpConstant/OpSpecConstant* → 常量与特化常量               │
└──────────────────────────┬─────────────────────────────────┘
                           ▼
┌─ Pass 3：注解回填（最关键的反向指针阶段）──────────────────┐
│  遍历 ⑨ 段的所有 OpDecorate / OpMemberDecorate：            │
│    DescriptorSet/Binding → 填 binding->set / ->binding     │
│    Location/Component    → 填 interface variable           │
│    Offset/MatrixStride   → 填 block 成员                    │
│    Block/BufferBlock     → 决定 UBO vs SSBO                 │
│    BuiltIn               → 标记内建量                        │
│    ArrayStride           → 填 array.stride                  │
│  遍历 ⑧ 段的 OpName / OpMemberName：填 name 字段            │
└──────────────────────────┬─────────────────────────────────┘
                           ▼
┌─ Pass 4：后处理 ───────────────────────────────────────────┐
│  按 set 分组描述符 → SpvReflectDescriptorSet               │
│  计算 block 的 size / padded_size / absolute_offset        │
│  建立入口点 → 资源的引用关系（读 OpEntryPoint interface）    │
│  记录 word_offset（用于 Change* 系列原地改写）              │
└────────────────────────────────────────────────────────────┘
```

### 7.1 为什么需要 `word_offset`

`SpvReflectDescriptorBinding` 里有个看似奇怪的字段：

```c
struct {
  uint32_t binding;
  uint32_t set;
} word_offset;
```

它记录 `OpDecorate ... Binding N` 这条指令中，**字面量 N 在指令流中的字偏移**。有了它，`spvReflectChangeDescriptorBindingNumbers()` 就能**原地打补丁**而不必重新解析整个模块。这也是 SPIRV-Reflect 能在运行时做"绑定重映射"的实现基础。

### 7.2 一个最小手写解析器（理解用，别上生产）

```cpp
struct SpvInstruction {
    uint32_t  wordCount() const { return words[0] >> 16; }
    uint32_t  opcode()    const { return words[0] & 0xFFFF; }
    uint32_t  operand(uint32_t i) const { return words[1 + i]; }
    const uint32_t* words;
};

void ScanDecorations(const std::vector<uint32_t>& spv) {
    if (spv[0] != 0x07230203) throw std::runtime_error("not a SPIR-V module");

    size_t i = 5;  // 跳过 5 字 header
    while (i < spv.size()) {
        SpvInstruction ins{ spv.data() + i };
        const uint32_t op = ins.opcode();

        if (op == SpvOpDecorate) {
            uint32_t target = ins.operand(0);
            uint32_t deco   = ins.operand(1);
            if (deco == SpvDecorationDescriptorSet)
                printf("id=%u  DescriptorSet=%u\n", target, ins.operand(2));
            else if (deco == SpvDecorationBinding)
                printf("id=%u  Binding=%u\n", target, ins.operand(2));
        }
        else if (op == SpvOpMemberDecorate) {
            uint32_t type = ins.operand(0), member = ins.operand(1), deco = ins.operand(2);
            if (deco == SpvDecorationOffset)
                printf("type=%u member=%u Offset=%u\n", type, member, ins.operand(3));
        }
        else if (op == SpvOpName) {
            printf("id=%u name=%s\n", ins.operand(0),
                   reinterpret_cast<const char*>(&ins.words[2]));
        }

        i += ins.wordCount();     // ★ 必须用 WordCount 跳，不能按操作数个数
    }
}
```

> 这 30 行能跑，但要上生产你得再处理：字符串跨字、工作组装饰、`OpDecorateId`、DecorationGroup、`OpSpecConstantOp`、递归类型、执行模式、非语义指令……**这些加起来就是 SPIRV-Reflect 那 6000 行的价值**。

---

## 8. 反射的边界：能拿到 vs 拿不到

### 8.1 能稳定拿到 ✅

| 信息 | 可靠性 | 备注 |
|------|-------|------|
| set / binding / descriptor type | ⭐⭐⭐ | 由规范强制，100% 可靠 |
| Block 成员 offset / size / stride / 主序 | ⭐⭐⭐ | 唯一真相 |
| 输入输出 location / component / format | ⭐⭐⭐ | — |
| PushConstant 范围（offset + size） | ⭐⭐⭐ | — |
| 入口点名、执行模型、LocalSize | ⭐⭐⭐ | — |
| 特化常量 ID 与默认值 | ⭐⭐⭐ | — |
| 数组维度、运行时长度标记 | ⭐⭐⭐ | — |
| **名字**（`name` / `semantic`） | ⭐⭐ | **依赖调试信息**，见下 |
| 能力集 / 扩展 | ⭐⭐⭐ | 用于校验设备支持 |

### 8.2 拿不到或不完全可靠 ⚠️

| 信息 | 原因 |
|------|------|
| **变量名** | 来自 `OpName`（调试信息）。一旦 `spirv-opt --strip-debug`，`name` 变 `NULL` 或空串 |
| **HLSL 语义名** | `semantic` 字段依赖 DXC 的调试信息；GLSL 编译通常没有 |
| 宏名 | 预处理阶段即消失 |
| 源码结构（if / for / 函数） | 已降为 CFG |
| 未被使用的静态函数 | 可能被 DCE |
| 源码文件路径 | `OpSource` 可被剥离；且会导致构建不可重现 |
| **"这是 PerFrame 还是 PerMaterial"** | SPIR-V 完全没有这个概念 → **必须靠命名约定或额外元数据**（见 06 篇） |
| 注释 | 除非以 `OpString` 形式保留 |

### 8.3 关于 `--strip-debug`：一个真实事故

```
现象：Debug 构建一切正常，Release 构建所有 set 编号错乱。
原因：Release 跑了 spirv-opt -O（隐含 strip debug），
      name 全部丢失，基于名字的 set 分配逻辑 fallback 到默认，全部落到 set 0。
```

**对策**（三选一）：
1. 反射的 set/binding **只依赖 `OpDecorate`，不依赖名字**（名字仅用于日志与查表优化）
2. 构建时保留名字：`spirv-opt --strip-nonsemantic` 而非完整 strip，或用 `-g` 生成 NonSemantic 调试信息
3. 在 Release 构建中**断言名字存在**，让问题在构建期暴露而非运行期

---

## 9. 校验：反射之前先验证

反射只能解析"语法上合法"的 SPIR-V，不能保证它满足 Vulkan 环境规则。必须在反射前跑：

```bash
# 按目标环境校验（会检查 Vulkan 特有的限制，如 interface 列表完整性）
spirv-val --target-env vulkan1.3 shader.frag.spv

# 常用组合
spirv-val --target-env vulkan1.3 --relax-block-layout off shader.spv
```

| 工具 | 作用 |
|------|------|
| `spirv-as` / `spirv-dis` | 汇编 / 反汇编（人肉排查神器） |
| `spirv-val` | 合法性 + 环境规则校验 |
| `spirv-opt` | 优化、剥离、重编号（`--legalize-hlsl`、`--strip-debug`） |
| `spirv-link` | 多模块链接 |
| `spirv-cross` | SPIR-V → GLSL/HLSL/MSL 反编译 |

> **工程建议**：把 `spirv-val` 接进构建流程，失败即中断。SPIRV-Reflect 对非法模块只会返回 `SPV_REFLECT_RESULT_ERROR_PARSE_FAILED`，不会告诉你是哪条规则违反了。

---

## 10. 小结

| 要点 | 内容 |
|------|------|
| 反射的物理基础 | Decoration 外置 → 只读 ⑤⑥⑧⑨⑩ 段即可 |
| 描述符类型 | 靠 **storage class + 类型构造**推导，没有显式声明 |
| 内存布局 | 来自 `OpMemberDecorate Offset/MatrixStride/RowMajor` |
| 入口点过滤 | 依赖 `OpEntryPoint` interface 列表，**SPIR-V ≥1.4 才包含描述符** |
| 名字 | 来自调试信息，**可被 strip 掉**，不能作为唯一真相 |
| 语义 | SPIR-V 不含"这是 PerFrame"这类语义，需约定 |
| 校验 | 反射前必须 `spirv-val` |

---

*上一篇：[00-总览与编译流水线](00-总览与编译流水线.md) | 下一篇：[02-SPIRV-Reflect 库详解与实战](02-SPIRV-Reflect库详解与实战.md)*
