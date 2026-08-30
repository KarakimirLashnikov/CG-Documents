# SPIRV-Reflect 库详解与实战

> 系列第 **2** 篇，也是整套系列的**工具核心**。SPIRV-Reflect 是 Khronos 官方维护的轻量 SPIR-V 反射库：单文件 C 实现、零依赖、只读解析，被大量引擎与框架用于自动生成 Vulkan 描述符布局。本文覆盖构建、数据结构、全部 API、七个实战场景与一份坑清单。

---

## 1. 库定位与设计哲学

### 1.1 它是什么

| 维度 | 描述 |
|------|------|
| 维护方 | Khronos Group（SPIRV-Reflect 官方仓库） |
| 语言 | C99（头文件带 `extern "C"`，C++ 直接可用） |
| 依赖 | **零**。只需要标准库 |
| 体积 | `spirv_reflect.c` + `spirv_reflect.h`，约 6000 行 |
| 能力 | 只读解析 + **有限的原地改写**（绑定号重映射） |
| 不做的事 | 不翻译着色器语言、不优化、不做完整校验 |

### 1.2 设计取舍（为什么它是这个样子）

```
┌──────────────────────────────────────────────────────┐
│  SPIRV-Reflect 的三条自我约束                          │
├──────────────────────────────────────────────────────┤
│  1. 不解析函数体  →  极快，且不随 shader 复杂度变慢       │
│  2. 只做一次线性扫描 + 少量回填  →  无 AST、无 IR 开销    │
│  3. 枚举值刻意与 Vulkan 对齐  →  static_cast 直接可用    │
└──────────────────────────────────────────────────────┘
```

第三条是工程上最舒服的一点：头文件里每个枚举值都带 `// = VK_XXX` 注释，说明**数值与 Vulkan 头完全一致**：

```cpp
// 这些都是合法且被推荐的写法
auto vkStage  = static_cast<VkShaderStageFlags>(module.shader_stage);
auto vkType   = static_cast<VkDescriptorType>(binding.descriptor_type);
auto vkFormat = static_cast<VkFormat>(var->format);
```

> ⚠️ 这是**依赖头文件约定的行为**，不是 ABI 保证。升级 SPIRV-Reflect 时应在 CI 里加一条静态断言（见 §11.4）。

---

## 2. 获取与构建

### 2.1 仓库结构

```
SPIRV-Reflect/
├── spirv_reflect.h                    ← 唯一需要包含的头文件
├── spirv_reflect.c                    ← 唯一需要编译的源文件
├── include/spirv/unified1/spirv.h     ← SPIR-V 官方枚举（依赖，随仓库分发）
├── main.cpp                           ← 可选：命令行工具 spirv-reflect
├── util/                              ← 可选：示例与辅助代码
└── CMakeLists.txt
```

### 2.2 三种集成方式

**方式 A：源码直拷（最省心，推荐给自研引擎）**

```bash
cp spirv_reflect.h spirv_reflect.c  <your-engine>/thirdparty/spirv-reflect/
cp -r include/spirv/unified1/       <your-engine>/thirdparty/spirv-reflect/include/spirv/
```

```cmake
add_library(spirv_reflect STATIC
    thirdparty/spirv-reflect/spirv_reflect.c)
target_include_directories(spirv_reflect PUBLIC
    thirdparty/spirv-reflect)
```

**方式 B：vcpkg**

```bash
vcpkg install spirv-reflect
```

```cmake
find_package(spirv-reflect CONFIG REQUIRED)
target_link_libraries(my_engine PRIVATE unofficial::spirv-reflect)
```

**方式 C：CMake FetchContent**

```cmake
include(FetchContent)
FetchContent_Declare(spirv_reflect
    GIT_REPOSITORY https://github.com/KhronosGroup/SPIRV-Reflect.git
    GIT_TAG        main)
FetchContent_MakeAvailable(spirv_reflect)
target_link_libraries(my_engine PRIVATE spirv-reflect-static)
```

### 2.3 编译注意事项

- 库是 C 文件，**不要**用 C++ 编译器直接编 `.c`（会触发 `extern "C"` 问题）。CMake 中放独立 target 最稳妥
- 头文件里 `__cplusplus` 分支已经处理了 `extern "C"`，C++ 侧直接 `#include "spirv_reflect.h"` 即可
- 支持 `-fno-exceptions -fno-rtti` 环境；内部用 `calloc/free`，可替换分配器（改 `#define`）
- **废弃符号**：`spvReflectGetShaderModule`、`spvReflectEnumeratePushConstants`、`spvReflectGetInputVariable`、`spvReflectGetOutputVariable`、`spvReflectGetPushConstant`、`spvReflectChangeDescriptorBindingNumber` 已标 `SPV_REFLECT_DEPRECATED`，新代码用替代版本

---

## 3. 数据结构全景

### 3.1 顶层：`SpvReflectShaderModule`

```c
typedef struct SpvReflectShaderModule {
  const char*                    generator;              // 未使用（保留）
  const char*                    entry_point_name;       // 第一个入口点名
  uint32_t                       entry_point_id;
  uint32_t                       entry_point_count;      // ★ 多入口点模块
  SpvReflectEntryPoint*          entry_points;

  SpvSourceLanguage              source_language;        // HLSL / GLSL / ...
  uint32_t                       source_language_version;
  const char*                    source_file;
  const char*                    source_source;          // 源码文本（若保留）

  uint32_t                       capability_count;
  SpvCapability*                 capabilities;

  SpvExecutionModel              spirv_execution_model;
  SpvReflectShaderStageFlagBits  shader_stage;           // ★ 与 VkShaderStageFlags 对齐

  uint32_t                       descriptor_binding_count;
  SpvReflectDescriptorBinding*   descriptor_bindings;
  uint32_t                       descriptor_set_count;
  SpvReflectDescriptorSet        descriptor_sets[SPV_REFLECT_MAX_DESCRIPTOR_SETS]; // 64

  uint32_t                       input_variable_count;
  SpvReflectInterfaceVariable**  input_variables;
  uint32_t                       output_variable_count;
  SpvReflectInterfaceVariable**  output_variables;
  uint32_t                       interface_variable_count;
  SpvReflectInterfaceVariable**  interface_variables;

  uint32_t                       push_constant_block_count;
  SpvReflectBlockVariable**      push_constant_blocks;

  uint32_t                       spec_constant_count;
  SpvReflectSpecializationConstant** spec_constants;

  struct { /* 内部状态，勿触碰 */ } _internal;
} SpvReflectShaderModule;
```

> **注意不对称**：`descriptor_bindings` 是**结构体数组**（值语义），而 `input_variables` / `push_constant_blocks` / `spec_constants` 是**指针数组**。写代码时容易搞混。

### 3.2 描述符：`SpvReflectDescriptorBinding`

```c
typedef struct SpvReflectDescriptorBinding {
  uint32_t                            spirv_id;
  const char*                         name;                  // ⚠ 依赖调试信息
  uint32_t                            binding;
  uint32_t                            input_attachment_index;
  uint32_t                            set;
  SpvReflectDescriptorType            descriptor_type;       // → VkDescriptorType
  SpvReflectResourceType              resource_type;         // Sampler/CBV/SRV/UAV
  SpvReflectImageTraits               image;                 // dim/depth/arrayed/ms/sampled/format
  SpvReflectBlockVariable             block;                 // UBO/SSBO 布局（递归）
  SpvReflectBindingArrayTraits        array;                 // dims[32] / dims_count
  uint32_t                            count;                 // ★ 描述符数组元素数（非数组为 1）
  uint32_t                            accessed;              // 是否被静态访问
  uint32_t                            uav_counter_id;        // HLSL counter buffer 的 ID
  struct SpvReflectDescriptorBinding* uav_counter_binding;   // ★ 指向对应的 counter binding
  uint32_t                            byte_address_buffer_offset_count;
  uint32_t*                           byte_address_buffer_offsets;
  SpvReflectTypeDescription*          type_description;
  struct { uint32_t binding; uint32_t set; } word_offset;    // 供 Change* 原地改写
  SpvReflectDecorationFlags           decoration_flags;
  SpvReflectUserType                  user_type;             // 需 SPV_GOOGLE_user_type
} SpvReflectDescriptorBinding;
```

**字段解读（踩坑高发区）**：

| 字段 | 说明 | 坑 |
|------|------|-----|
| `count` | 描述符数组元素数；**非数组为 1**；运行时长度数组可能为 **0** | 直接当 descriptorCount 用会创建 0 大小的绑定 |
| `block` | UBO/SSBO 才有意义；Sampler/Image 时内容为空 | 别对 Image 读 `block.members` |
| `array` | 数组维度。`dims_count=0` 表示非数组；`dims[0]=0` 通常是运行时长度数组 | Bindless 判断依据 |
| `uav_counter_binding` | HLSL `AppendStructuredBuffer` / `RWStructuredBuffer` 带 counter 时，DXC 会**额外生成一个 counter buffer 绑定** | 描述符布局会"凭空多出一个 binding"，必须一并处理 |
| `accessed` | 静态分析结果：该资源是否被实际访问 | 用于剔除 uber shader 中未使用的绑定 |
| `decoration_flags` | `NON_WRITABLE`、`NON_READABLE`、`BLOCK`、`BUFFER_BLOCK`、`ROW_MAJOR` 等 | `NON_WRITABLE` 常见于 `Texture2D`（只读 SRV） |

### 3.3 块变量：`SpvReflectBlockVariable`

```c
typedef struct SpvReflectBlockVariable {
  uint32_t                    spirv_id;
  const char*                 name;
  uint32_t                    offset;            // 相对父结构
  uint32_t                    absolute_offset;   // ★ 相对块起点（生成 C++ 结构体用这个）
  uint32_t                    size;              // 实际字节数（vec3 = 12）
  uint32_t                    padded_size;       // std140/std430 填充后大小（vec3 = 16）
  SpvReflectDecorationFlags   decoration_flags;
  SpvReflectNumericTraits     numeric;           // scalar/vector/matrix 细节
  SpvReflectArrayTraits       array;             // dims[32] + stride
  uint32_t                    flags;
  uint32_t                    member_count;
  SpvReflectBlockVariable*    members;           // 递归展开嵌套结构
  SpvReflectTypeDescription*  type_description;
} SpvReflectBlockVariable;
```

`SpvReflectNumericTraits` 的结构：

```c
typedef struct SpvReflectNumericTraits {
  struct { uint32_t width; uint32_t signedness; } scalar;
  struct { uint32_t component_count;           } vector;
  struct { uint32_t column_count; uint32_t row_count; uint32_t stride; } matrix;
} SpvReflectNumericTraits;
```

### 3.4 接口变量与入口点

```c
typedef struct SpvReflectInterfaceVariable {
  uint32_t                      spirv_id;
  const char*                   name;
  uint32_t                      location;
  uint32_t                      component;
  SpvStorageClass               storage_class;
  const char*                   semantic;          // HLSL 语义（可为空）
  SpvReflectDecorationFlags     decoration_flags;
  SpvBuiltIn                    built_in;          // ★ 非 0 = 内建量
  SpvReflectNumericTraits       numeric;
  SpvReflectArrayTraits         array;
  uint32_t                      member_count;
  SpvReflectInterfaceVariable*  members;
  SpvReflectFormat              format;            // → VkFormat
  SpvReflectTypeDescription*    type_description;
} SpvReflectInterfaceVariable;

typedef struct SpvReflectEntryPoint {
  const char*                   name;
  uint32_t                      id;
  SpvExecutionModel             spirv_execution_model;
  SpvReflectShaderStageFlagBits shader_stage;
  uint32_t                      input_variable_count;   ...input_variables
  uint32_t                      output_variable_count;  ...output_variables
  uint32_t                      interface_variable_count; ...interface_variables
  uint32_t                      descriptor_set_count;   ...descriptor_sets
  uint32_t                      used_uniform_count;     ...used_uniforms
  uint32_t                      used_push_constant_count; ...used_push_constants
  uint32_t                      execution_mode_count;   ...execution_modes
  struct { uint32_t x, y, z; }  local_size;       // ★ 计算着色器
  uint32_t                      invocations;      // tess / geometry
  uint32_t                      output_vertices;
  uint32_t                      resource_heap_access_count;  ...resource_heap_accesses
  uint32_t                      sampler_heap_access_count;   ...sampler_heap_accesses
} SpvReflectEntryPoint;
```

### 3.5 特化常量

```c
typedef struct SpvReflectSpecializationConstant {
  uint32_t                     spirv_id;
  uint32_t                     constant_id;        // ← 对应 VkSpecializationMapEntry::constantID
  const char*                  name;
  SpvReflectTypeDescription*   type_description;
  uint32_t                     default_value_size; // 4 或 8 字节（对齐到 4）
  void*                        default_value;      // 原始位模式
} SpvReflectSpecializationConstant;
```

`default_value` 的解释规则（头文件原文语义）：

| `type_description->op` | `default_value_size` | `default_value` 内容 |
|------------------------|---------------------|----------------------|
| `SpvOpSpecConstantTrue` | 4 | `uint32_t(1)` |
| `SpvOpSpecConstantFalse` | 4 | `uint32_t(0)` |
| `SpvOpSpecConstant` | 4（≤32 位）/ 8（64 位） | 标量的位模式；**低位字在前** |

---

## 4. API 全景

### 4.1 生命周期

| 函数 | 作用 |
|------|------|
| `spvReflectCreateShaderModule(size, p_code, p_module)` | 解析；**默认拷贝** SPIR-V |
| `spvReflectCreateShaderModule2(flags, size, p_code, p_module)` | 同上，可传 `SPV_REFLECT_MODULE_FLAG_NO_COPY` |
| `spvReflectDestroyShaderModule(p_module)` | 释放；**必须调用** |
| `spvReflectGetCodeSize(p_module)` | 取回模块持有的 SPIR-V 字节数 |
| `spvReflectGetCode(p_module)` | 取回 SPIR-V 指针（改写后从这里拿） |

### 4.2 枚举（"两遍调用"模式）

所有 `Enumerate` 函数统一遵循：**第一遍传 `nullptr` 取数量，第二遍传缓冲区**。

```c
SpvReflectResult spvReflectEnumerateDescriptorSets(
    const SpvReflectShaderModule* p_module,
    uint32_t*                     p_count,
    SpvReflectDescriptorSet**     pp_sets);     // nullptr → 只填 p_count
```

| 函数 | 返回元素类型 | 范围 |
|------|-------------|------|
| `EnumerateDescriptorBindings` | `SpvReflectDescriptorBinding*` | 模块全部绑定 |
| `EnumerateDescriptorSets` | `SpvReflectDescriptorSet*` | 模块全部 set |
| `EnumerateEntryPointDescriptorBindings` | 同上 | **仅该入口点使用** |
| `EnumerateEntryPointDescriptorSets` | 同上 | 同上 |
| `EnumerateInterfaceVariables` | `SpvReflectInterfaceVariable*` | Input + Output |
| `EnumerateInputVariables` / `EnumerateOutputVariables` | 同上 | 单独一侧 |
| `EnumerateEntryPoint{Input,Output,Interface}Variables` | 同上 | 按入口点过滤 |
| `EnumeratePushConstantBlocks` | `SpvReflectBlockVariable*` | 全部 |
| `EnumerateEntryPointPushConstantBlocks` | 同上 | 按入口点过滤 |
| `EnumerateSpecializationConstants` | `SpvReflectSpecializationConstant*` | 全部 |

### 4.3 精确查询

| 函数 | 按键 |
|------|------|
| `spvReflectGetDescriptorBinding(m, binding, set, &result)` | `(set, binding)` |
| `spvReflectGetEntryPointDescriptorBinding(m, entry, binding, set, &result)` | 入口点 + `(set, binding)` |
| `spvReflectGetDescriptorSet(m, set, &result)` | set 号 |
| `spvReflectGetInputVariableByLocation(m, loc, &result)` | location |
| `spvReflectGetInputVariableBySemantic(m, "TEXCOORD0", &result)` | HLSL 语义 |
| `spvReflectGetOutputVariableBy{Location,Semantic}` | 同上（输出侧） |
| `spvReflectGetPushConstantBlock(m, index, &result)` | 索引 |
| `spvReflectGetEntryPoint(m, "PSMain")` | 入口点名 |
| `spvReflectSourceLanguage(SpvSourceLanguage)` | 转字符串 |

### 4.4 原地改写（绑定重映射）

| 函数 | 作用 |
|------|------|
| `spvReflectChangeDescriptorBindingNumbers(m, binding, newBinding, newSet)` | 改单个绑定的 set/binding |
| `spvReflectChangeDescriptorSetNumber(m, set, newSet)` | 整个 set 换号 |
| `spvReflectChangeInputVariableLocation(m, var, newLoc)` | 改输入 location |
| `spvReflectChangeOutputVariableLocation(m, var, newLoc)` | 改输出 location |

**"不改"哨兵值**：

```c
SPV_REFLECT_BINDING_NUMBER_DONT_CHANGE = ~0;
SPV_REFLECT_SET_NUMBER_DONT_CHANGE     = ~0;
```

### 4.5 返回值

```c
typedef enum SpvReflectResult {
  SPV_REFLECT_RESULT_SUCCESS,
  SPV_REFLECT_RESULT_NOT_READY,
  SPV_REFLECT_RESULT_ERROR_PARSE_FAILED,
  SPV_REFLECT_RESULT_ERROR_ALLOC_FAILED,
  SPV_REFLECT_RESULT_ERROR_RANGE_EXCEEDED,
  SPV_REFLECT_RESULT_ERROR_NULL_POINTER,
  SPV_REFLECT_RESULT_ERROR_INTERNAL_ERROR,
  SPV_REFLECT_RESULT_ERROR_COUNT_MISMATCH,
  SPV_REFLECT_RESULT_ERROR_ELEMENT_NOT_FOUND,
  SPV_REFLECT_RESULT_ERROR_SPIRV_INVALID_CODE_SIZE,
  SPV_REFLECT_RESULT_ERROR_SPIRV_INVALID_MAGIC_NUMBER,
  SPV_REFLECT_RESULT_ERROR_SPIRV_UNEXPECTED_EOF,
  SPV_REFLECT_RESULT_ERROR_SPIRV_INVALID_ID_REFERENCE,
  SPV_REFLECT_RESULT_ERROR_SPIRV_SET_NUMBER_OVERFLOW,   // ← set ≥ 64
  SPV_REFLECT_RESULT_ERROR_SPIRV_INVALID_STORAGE_CLASS,
  SPV_REFLECT_RESULT_ERROR_SPIRV_RECURSION,
  SPV_REFLECT_RESULT_ERROR_SPIRV_INVALID_INSTRUCTION,
  SPV_REFLECT_RESULT_ERROR_SPIRV_UNEXPECTED_BLOCK_DATA,
  SPV_REFLECT_RESULT_ERROR_SPIRV_INVALID_BLOCK_MEMBER_REFERENCE,
  SPV_REFLECT_RESULT_ERROR_SPIRV_INVALID_ENTRY_POINT,
  SPV_REFLECT_RESULT_ERROR_SPIRV_INVALID_EXECUTION_MODE,
  SPV_REFLECT_RESULT_ERROR_SPIRV_MAX_RECURSIVE_EXCEEDED,
} SpvReflectResult;
```

---

## 5. 实战一：最小加载与错误处理

```cpp
// ShaderReflection.h
#pragma once
#include <cstdint>
#include <vector>
#include <stdexcept>
#include <string>
#include <format>
#include "spirv_reflect.h"

namespace rt {

inline void CheckSpvReflect(SpvReflectResult r, std::string_view where) {
    if (r != SPV_REFLECT_RESULT_SUCCESS)
        throw std::runtime_error(std::format("SPIRV-Reflect failed @ {}: code {}",
                                             where, static_cast<int>(r)));
}

class ShaderModule {
public:
    explicit ShaderModule(std::vector<uint32_t> spv)
        : spv_(std::move(spv)) {
        // 注意：size 参数是【字节数】，不是字数
        auto r = spvReflectCreateShaderModule(spv_.size() * sizeof(uint32_t),
                                              spv_.data(), &module_);
        CheckSpvReflect(r, "spvReflectCreateShaderModule");
    }

    ~ShaderModule() { spvReflectDestroyShaderModule(&module_); }

    ShaderModule(const ShaderModule&)            = delete;
    ShaderModule& operator=(const ShaderModule&) = delete;

    const SpvReflectShaderModule* get() const { return &module_; }

    // 两遍调用模式的通用封装
    template <typename T, typename Fn>
    std::vector<T*> Enumerate(Fn&& fn) const {
        uint32_t count = 0;
        CheckSpvReflect(fn(&module_, &count, nullptr), "enumerate(count)");
        std::vector<T*> out(count);
        if (count)
            CheckSpvReflect(fn(&module_, &count, out.data()), "enumerate(fill)");
        return out;
    }

private:
    std::vector<uint32_t>   spv_;
    SpvReflectShaderModule  module_{};
};

} // namespace rt
```

使用：

```cpp
auto spv = ReadFileBinary("assets/shaders/pbr.frag.spv");
rt::ShaderModule mod(std::move(spv));

printf("stage  = 0x%x\n", mod.get()->shader_stage);
printf("entry  = %s\n",   mod.get()->entry_point_name);
printf("lang   = %s\n",   spvReflectSourceLanguage(mod.get()->source_language));
```

> ⚠️ **`size` 参数极易写错**：是**字节数**（`words * 4`）。传错会得到 `SPV_REFLECT_RESULT_ERROR_SPIRV_INVALID_CODE_SIZE` 或直接崩。

---

## 6. 实战二：枚举描述符集并打印完整绑定表

```cpp
static const char* DescribeType(SpvReflectDescriptorType t) {
    switch (t) {
    case SPV_REFLECT_DESCRIPTOR_TYPE_SAMPLER:                       return "SAMPLER";
    case SPV_REFLECT_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER:        return "COMBINED_IMAGE_SAMPLER";
    case SPV_REFLECT_DESCRIPTOR_TYPE_SAMPLED_IMAGE:                 return "SAMPLED_IMAGE";
    case SPV_REFLECT_DESCRIPTOR_TYPE_STORAGE_IMAGE:                 return "STORAGE_IMAGE";
    case SPV_REFLECT_DESCRIPTOR_TYPE_UNIFORM_TEXEL_BUFFER:          return "UNIFORM_TEXEL_BUFFER";
    case SPV_REFLECT_DESCRIPTOR_TYPE_STORAGE_TEXEL_BUFFER:          return "STORAGE_TEXEL_BUFFER";
    case SPV_REFLECT_DESCRIPTOR_TYPE_UNIFORM_BUFFER:                return "UNIFORM_BUFFER";
    case SPV_REFLECT_DESCRIPTOR_TYPE_STORAGE_BUFFER:                return "STORAGE_BUFFER";
    case SPV_REFLECT_DESCRIPTOR_TYPE_UNIFORM_BUFFER_DYNAMIC:        return "UNIFORM_BUFFER_DYNAMIC";
    case SPV_REFLECT_DESCRIPTOR_TYPE_STORAGE_BUFFER_DYNAMIC:        return "STORAGE_BUFFER_DYNAMIC";
    case SPV_REFLECT_DESCRIPTOR_TYPE_INPUT_ATTACHMENT:              return "INPUT_ATTACHMENT";
    case SPV_REFLECT_DESCRIPTOR_TYPE_ACCELERATION_STRUCTURE_KHR:    return "ACCELERATION_STRUCTURE";
    default:                                                        return "?";
    }
}

void DumpBindings(const rt::ShaderModule& m) {
    auto sets = m.Enumerate<SpvReflectDescriptorSet>(spvReflectEnumerateDescriptorSets);

    for (auto* set : sets) {
        printf("=== set %u  (%u bindings) ===\n", set->set, set->binding_count);
        for (uint32_t i = 0; i < set->binding_count; ++i) {
            const auto& b = *set->bindings[i];

            // 数组信息
            std::string arrInfo;
            for (uint32_t d = 0; d < b.array.dims_count; ++d)
                arrInfo += std::format("[{}]", b.array.dims[d]);

            printf("  binding %-3u %-26s name=%-24s count=%-3u stage=0x%x accessed=%u %s\n",
                   b.binding,
                   DescribeType(b.descriptor_type),
                   b.name ? b.name : "<noname>",
                   b.count,
                   static_cast<unsigned>(m.get()->shader_stage),
                   b.accessed,
                   arrInfo.c_str());

            // UBO / SSBO：递归打印成员布局
            if (b.descriptor_type == SPV_REFLECT_DESCRIPTOR_TYPE_UNIFORM_BUFFER ||
                b.descriptor_type == SPV_REFLECT_DESCRIPTOR_TYPE_STORAGE_BUFFER) {
                DumpBlock(b.block, 4);
            }

            // HLSL counter buffer：DXC 为 Append/Consume/RW StructuredBuffer 额外生成
            if (b.uav_counter_binding)
                printf("    %-6s counter buffer -> set=%u binding=%u\n", "",
                       b.uav_counter_binding->set, b.uav_counter_binding->binding);
        }
    }
}

void DumpBlock(const SpvReflectBlockVariable& v, int indent) {
    const std::string pad(indent, ' ');
    printf("%s%-24s off=%-5u abs=%-5u size=%-5u padded=%-5u flags=0x%x\n",
           pad.c_str(), v.name ? v.name : "?", v.offset, v.absolute_offset,
           v.size, v.padded_size, v.decoration_flags);

    // 数组 / 矩阵细节
    if (v.array.dims_count)
        printf("%s  array dims=%u stride=%u\n", pad.c_str(), v.array.dims[0], v.array.stride);
    if (v.numeric.matrix.column_count)
        printf("%s  matrix %ux%u stride=%u %s\n", pad.c_str(),
               v.numeric.matrix.column_count, v.numeric.matrix.row_count,
               v.numeric.matrix.stride,
               (v.decoration_flags & SPV_REFLECT_DECORATION_ROW_MAJOR) ? "RowMajor" : "ColMajor");

    for (uint32_t i = 0; i < v.member_count; ++i)
        DumpBlock(v.members[i], indent + 2);
}
```

**典型输出**：

```
=== set 0  (3 bindings) ===
  binding 0   UNIFORM_BUFFER            name=PerFrame               count=1   stage=0x10 accessed=1
    PerFrame                 off=0    abs=0    size=80   padded=80  flags=0x1
      viewProj              off=0    abs=0    size=64   padded=64  flags=0x0
        array dims=0 stride=16
        matrix 4x4 stride=16 RowMajor
      cameraPos             off=64   abs=64   size=16   padded=16  flags=0x0
  binding 1   SAMPLED_IMAGE             name=albedoTex              count=1   stage=0x10 accessed=1
  binding 2   SAMPLER                   name=linearSampler          count=1   stage=0x10 accessed=1
```

> 用这个 dump 函数配合 `spirv-dis`，几乎可以定位所有"绑定对不上"的问题。

---

## 7. 实战三：Push Constant

```cpp
struct PushConstantRangeInfo {
    uint32_t offset;
    uint32_t size;
    VkShaderStageFlags stages;
};

std::vector<PushConstantRangeInfo> CollectPushConstants(const rt::ShaderModule& m) {
    auto blocks = m.Enumerate<SpvReflectBlockVariable>(spvReflectEnumeratePushConstantBlocks);

    std::vector<PushConstantRangeInfo> out;
    for (auto* b : blocks) {
        out.push_back({
            .offset = b->offset,
            .size   = b->size,          // ★ 用 size 还是 padded_size？
            .stages = static_cast<VkShaderStageFlags>(m.get()->shader_stage),
        });

        // 打印成员，便于生成 C++ 结构体
        for (uint32_t i = 0; i < b->member_count; ++i)
            printf("  pc member: %-20s offset=%u size=%u\n",
                   b->members[i].name, b->members[i].absolute_offset, b->members[i].size);
    }
    return out;
}
```

**`size` vs `padded_size` 怎么选**：

| 场景 | 用哪个 | 理由 |
|------|--------|------|
| `VkPushConstantRange::size` | `padded_size` | Vulkan 要求按 4 字节对齐；且 offset+size 必须覆盖整个块 |
| 生成 C++ 结构体的 `static_assert` | `size` | 反映真实数据大小 |

```cpp
// 安全做法：对 padded_size 向上对齐到 4
uint32_t AlignedSize = (b->padded_size + 3) & ~3u;
```

> ⚠️ **Vulkan 限制**：所有 PushConstant 的总大小通常只有 **128 字节**（`maxPushConstantsSize`），超过会导致管线创建失败。反射出多个 PushConstant block 时，必须**合并成一个连续区间**再交给 Vulkan：

```cpp
// 多个 block 时取 min(offset) .. max(offset+size) 的并集
uint32_t lo = UINT32_MAX, hi = 0;
for (auto& r : ranges) { lo = std::min(lo, r.offset); hi = std::max(hi, r.offset + r.size); }
VkPushConstantRange merged{ .offset = lo, .size = hi - lo };
```

---

## 8. 实战四：顶点输入布局

```cpp
std::vector<VkVertexInputAttributeDescription>
BuildVertexAttributes(const rt::ShaderModule& vs) {
    auto inputs = vs.Enumerate<SpvReflectInterfaceVariable>(spvReflectEnumerateInputVariables);

    std::vector<VkVertexInputAttributeDescription> attrs;
    for (auto* v : inputs) {
        // 1. 跳过内建量（SV_VertexID / gl_VertexIndex 等）
        if (v->built_in != SpvBuiltInMax) continue;
        if (v->decoration_flags & SPV_REFLECT_DECORATION_BUILT_IN) continue;

        // 2. 跳过数组（多数驱动对数组顶点属性支持有限）
        if (v->array.dims_count > 0) continue;

        // 3. location / component / format
        attrs.push_back(VkVertexInputAttributeDescription{
            .location = v->location,
            .binding  = 0,                                  // 由引擎的顶点缓冲策略决定
            .format   = static_cast<VkFormat>(v->format),   // 数值与 VkFormat 对齐
            .offset   = 0,                                  // 见下
        });

        printf("attr: %-16s loc=%u comp=%u fmt=%d semantic=%s\n",
               v->name, v->location, v->component, (int)v->format,
               v->semantic ? v->semantic : "-");
    }
    return attrs;
}
```

**offset 怎么定**：SPIR-V **不含**顶点缓冲的字节偏移（那是 CPU 侧的布局决策）。三种做法：

| 做法 | 适用 |
|------|------|
| 按语义约定硬编码表（`POSITION`→0, `NORMAL`→12, `TEXCOORD0`→24 ...） | 网格格式固定的引擎 |
| 按 location 顺序紧凑排列，累加 `format` 的字节数 | 通用管线 |
| 在 shader 中用 `[[vk::location(n)]]` 显式指定，CPU 侧用同一张表 | ⭐ 推荐，双向可控 |

```cpp
static uint32_t FormatSize(VkFormat f) {
    switch (f) {
    case VK_FORMAT_R32_SFLOAT:             return 4;
    case VK_FORMAT_R32G32_SFLOAT:          return 8;
    case VK_FORMAT_R32G32B32_SFLOAT:       return 12;
    case VK_FORMAT_R32G32B32A32_SFLOAT:    return 16;
    // ... 按需扩展
    default: throw std::runtime_error("unhandled vertex format");
    }
}

uint32_t off = 0;
std::sort(attrs.begin(), attrs.end(),
          [](auto& a, auto& b) { return a.location < b.location; });
for (auto& a : attrs) { a.offset = off; off += FormatSize(a.format); }
// off 就是顶点步幅
```

---

## 9. 实战五：特化常量

```cpp
struct SpecConstantInfo {
    uint32_t    constantId;
    std::string name;
    bool        isBool;
    // 默认值位模式
    uint64_t    defaultValue;
};

std::vector<SpecConstantInfo> CollectSpecConstants(const rt::ShaderModule& m) {
    auto consts = m.Enumerate<SpvReflectSpecializationConstant>(
        spvReflectEnumerateSpecializationConstants);

    std::vector<SpecConstantInfo> out;
    for (auto* c : consts) {
        SpecConstantInfo info{
            .constantId = c->constant_id,
            .name       = c->name ? c->name : "",
            .isBool     = c->type_description &&
                          (c->type_description->op == SpvOpSpecConstantTrue ||
                           c->type_description->op == SpvOpSpecConstantFalse),
            .defaultValue = 0,
        };
        // 读取默认值的原始位模式（低位字在前）
        if (c->default_value) {
            if (c->default_value_size == 4)
                info.defaultValue = *reinterpret_cast<const uint32_t*>(c->default_value);
            else if (c->default_value_size == 8)
                info.defaultValue = *reinterpret_cast<const uint64_t*>(c->default_value);
        }
        out.push_back(std::move(info));

        printf("spec id=%-3u name=%-20s size=%u default=0x%llx\n",
               info.constantId, info.name.c_str(), c->default_value_size,
               (unsigned long long)info.defaultValue);
    }
    return out;
}
```

配合 Vulkan 使用（这是**消除变体**的关键，见 [05 篇](05-可编程着色器元编程.md)）：

```cpp
// HLSL:
//   [[vk::constant_id(0)]] const bool ENABLE_TAA = true;
//   [[vk::constant_id(1)]] const int  SAMPLE_COUNT = 4;

std::vector<VkSpecializationMapEntry> entries;
std::vector<uint8_t> data;

// 用反射拿到的 constantId 填 entries，避免手写 ID 漂移
for (auto& sc : specConstants) {
    size_t off = data.size();
    // 追加你想覆盖的值（类型必须与 shader 中一致）
    // ...
    entries.push_back({ sc.constantId, /*offset=*/(uint32_t)off, /*size=*/4 });
}

VkSpecializationInfo specInfo{
    .mapEntryCount = (uint32_t)entries.size(),
    .pMapEntries   = entries.data(),
    .dataSize      = data.size(),
    .pData         = data.data(),
};
```

---

## 10. 实战六：多入口点模块与入口点级过滤

一个 `.spv` 可以包含多个入口点（例如把同一个库的 VS/PS/CS 编进一个模块，或用 `spirv-link` 合并）。

```cpp
void DumpEntryPoints(const rt::ShaderModule& m) {
    const auto* mod = m.get();
    printf("module has %u entry point(s)\n", mod->entry_point_count);

    for (uint32_t i = 0; i < mod->entry_point_count; ++i) {
        const auto& ep = mod->entry_points[i];
        printf("[%u] %-16s stage=0x%x sets=%u uniforms=%u pcs=%u local=(%u,%u,%u)\n",
               i, ep.name, ep.shader_stage, ep.descriptor_set_count,
               ep.used_uniform_count, ep.used_push_constant_count,
               ep.local_size.x, ep.local_size.y, ep.local_size.z);

        // 按名字查询
        const SpvReflectEntryPoint* p = spvReflectGetEntryPoint(mod, ep.name);
        assert(p == &ep);
    }
}
```

按入口点枚举资源（**过滤掉未被该入口点使用的绑定**）：

```cpp
std::vector<SpvReflectDescriptorBinding*>
GetBindingsForEntryPoint(const SpvReflectShaderModule* m, const char* entry) {
    uint32_t count = 0;
    CheckSpvReflect(
        spvReflectEnumerateEntryPointDescriptorBindings(m, entry, &count, nullptr),
        "EntryPointDescriptorBindings(count)");

    std::vector<SpvReflectDescriptorBinding*> out(count);
    if (count)
        CheckSpvReflect(
            spvReflectEnumerateEntryPointDescriptorBindings(m, entry, &count, out.data()),
            "EntryPointDescriptorBindings(fill)");
    return out;
}
```

> ⚠️ **版本陷阱**（见 [01 篇 §6.1](01-SPIR-V模块结构与反射原理.md#61-opentrypoint-的-interface-列表)）：该函数依赖 `OpEntryPoint` 的 interface 列表，**SPIR-V < 1.4 的模块可能返回 0 条**。防御性写法：

```cpp
auto bindings = GetBindingsForEntryPoint(mod, "PSMain");
if (bindings.empty()) {
    // 回退：用模块级枚举 + accessed 标记过滤
    uint32_t n = 0;
    spvReflectEnumerateDescriptorBindings(mod, &n, nullptr);
    std::vector<SpvReflectDescriptorBinding*> all(n);
    spvReflectEnumerateDescriptorBindings(mod, &n, all.data());
    for (auto* b : all) if (b->accessed) bindings.push_back(b);
}
```

---

## 11. 实战七：绑定重映射（把 shader 的绑定号改成引擎的）

这是 SPIRV-Reflect 的"隐藏大招"：**在运行时改写 SPIR-V 的 `OpDecorate`，无需重新编译**。

### 11.1 典型场景

| 场景 | 做法 |
|------|------|
| DXC 默认把所有资源放到 `space0` → 全挤在 set 0 | 按命名/类型规则把资源搬到不同的 set |
| 第三方 shader 的绑定号与引擎冲突 | 运行时重映射，不改源码 |
| 自动分配（auto-binding） | 反射 → 决策 → 改写 → 用改写后的 SPIR-V 建管线 |
| 实现"无限描述符" | 把所有资源改写进一个 bindless 大数组 |

### 11.2 单个绑定重映射

```cpp
void RemapBinding(SpvReflectShaderModule* m, uint32_t set, uint32_t binding,
                  uint32_t newSet, uint32_t newBinding) {
    SpvReflectResult r;
    const auto* b = spvReflectGetDescriptorBinding(m, binding, set, &r);
    if (r != SPV_REFLECT_RESULT_SUCCESS) throw std::runtime_error("binding not found");

    CheckSpvReflect(
        spvReflectChangeDescriptorBindingNumbers(m, b, newBinding, newSet),
        "ChangeDescriptorBindingNumbers");
}
```

### 11.3 整个 set 换号

```cpp
void MoveSet(SpvReflectShaderModule* m, uint32_t from, uint32_t to) {
    uint32_t count = 0;
    spvReflectEnumerateDescriptorSets(m, &count, nullptr);
    std::vector<SpvReflectDescriptorSet*> sets(count);
    spvReflectEnumerateDescriptorSets(m, &count, sets.data());

    for (auto* s : sets) {
        if (s->set == from) {
            CheckSpvReflect(spvReflectChangeDescriptorSetNumber(m, s, to),
                            "ChangeDescriptorSetNumber");
        }
    }
    // ⚠ 改写后内部数组会被重排，旧指针可能失效 → 必须重新枚举
}
```

### 11.4 取回改写后的 SPIR-V

```cpp
void SaveRemapped(SpvReflectShaderModule* m, const char* path) {
    uint32_t       size = spvReflectGetCodeSize(m);          // 字节数
    const uint32_t* code = spvReflectGetCode(m);

    // 改写了就必须用这份新的 SPIR-V 去创建 VkShaderModule！
    VkShaderModuleCreateInfo ci{
        .sType    = VK_STRUCTURE_TYPE_SHADER_MODULE_CREATE_INFO,
        .codeSize = size,
        .pCode    = code,
    };
    vkCreateShaderModule(device, &ci, nullptr, &outModule);
}
```

> ⚠️ **最致命的坑**：改写后**必须**用 `spvReflectGetCode()` 返回的 SPIR-V 创建 `VkShaderModule`。如果你用了改写前的原始 buffer，Vulkan 看到的还是旧绑定号，而你的 `DescriptorSetLayout` 是新号 → 描述符不匹配，行为未定义或直接校验层报错。

### 11.5 `NO_COPY` 的语义

```cpp
SpvReflectShaderModule m{};
spvReflectCreateShaderModule2(SPV_REFLECT_MODULE_FLAG_NO_COPY,
                              bytes, code, &m);
```

| 模式 | 内存行为 | 改写影响 |
|------|---------|---------|
| 默认（拷贝） | 库内部持有一份副本；你可以立刻释放原 buffer | 改写入**内部副本**，用 `GetCode` 取回 |
| `NO_COPY` | 模块**直接引用**你的缓冲区，不额外分配 | 改写入**你的缓冲区**（就地修改） |

`NO_COPY` 适用于"加载 → 反射 → 改写 → 立刻用"的一次性流程；长期持有模块则用默认模式更安全。

### 11.6 限制

- `set` 号必须 < `SPV_REFLECT_MAX_DESCRIPTOR_SETS`（**64**），否则返回 `SPV_REFLECT_RESULT_ERROR_SPIRV_SET_NUMBER_OVERFLOW`
- 数组维度上限 `SPV_REFLECT_MAX_ARRAY_DIMS`（**32**）
- 只能改 **Decoration 的字面量**，不能改类型、不能插入/删除指令
- 改的是 `OpDecorate`；如果该 Decoration 通过 `OpDecorationGroup` 共享给多个变量，**会影响所有共享者**

---

## 12. 内存与生命周期规则

### 12.1 所有权表

| 数据 | 谁分配 | 谁释放 | 生命周期 |
|------|--------|--------|---------|
| `SpvReflectShaderModule` 结构体 | 你（栈/堆） | 你 | — |
| 内部数组、字符串 | 库 | `spvReflectDestroyShaderModule` | 与模块同生共死 |
| `Enumerate` 返回的**指针数组** | **你** | **你**（`delete[]` / `vector`） | 用完即弃 |
| `Enumerate` 返回的**元素** | 库 | 库 | 模块销毁后**失效** |

### 12.2 铁律

```cpp
// ✅ 正确：直接用，用完就扔
auto sets = m.Enumerate<SpvReflectDescriptorSet>(spvReflectEnumerateDescriptorSets);
for (auto* s : sets) { /* ... */ }
// sets 析构，指针数组释放；指向的元素仍归库所有

// ❌ 错误：把 name / binding 指针缓存到模块销毁之后
const char* cachedName = nullptr;
{
    rt::ShaderModule m(spv);
    cachedName = m.get()->descriptor_bindings[0].name;   // 悬垂！
}
printf("%s", cachedName);   // UB

// ✅ 正确：拷成 std::string
std::string cachedName = m.get()->descriptor_bindings[0].name;
```

### 12.3 长期持有的推荐模式

不要长期持有 `SpvReflectShaderModule`。反射一次，把结果**拷进自己的 POD 结构**：

```cpp
struct BindingInfo {
    uint32_t    set, binding, count;
    VkDescriptorType type;
    VkShaderStageFlags stages;
    std::string name;
    bool        runtimeSized;
};

struct ShaderReflectionData {
    VkShaderStageFlags stage;
    std::vector<BindingInfo>       bindings;
    std::vector<PushConstantInfo>  pushConstants;
    std::vector<VertexAttribute>   attributes;
    // ...
};

ShaderReflectionData Bake(const rt::ShaderModule& m) {
    ShaderReflectionData d;
    d.stage = static_cast<VkShaderStageFlags>(m.get()->shader_stage);
    for (auto* b : m.Enumerate<SpvReflectDescriptorBinding>(spvReflectEnumerateDescriptorBindings)) {
        d.bindings.push_back({
            .set          = b->set,
            .binding      = b->binding,
            .count        = (b->count > 0) ? b->count : 1,
            .type         = static_cast<VkDescriptorType>(b->descriptor_type),
            .stages       = d.stage,
            .name         = b->name ? b->name : "",
            .runtimeSized = (b->count == 0),
        });
    }
    return d;   // 与 SPIRV-Reflect 彻底解耦
}
```

**好处**：
1. SPIRV-Reflect 只在加载期出现一次，运行时零依赖
2. `ShaderReflectionData` 可序列化 → 支持离线烘焙与加载加速
3. 换反射库（如改用 Slang）只改 `Bake()` 一个函数

---

## 13. 线程安全与性能

| 维度 | 结论 |
|------|------|
| 单个 `SpvReflectShaderModule` | **不可并发读写**。解析完成后只读访问是安全的（无内部可变状态） |
| 不同模块 | 完全独立，可并行 |
| `Change*` 系列 | 修改内部状态，**必须外部加锁** |
| 分配器 | 用标准 `calloc/free`，进程级线程安全 |

**性能数据（经验值，供规划参考）**：

| Shader 规模 | 解析耗时 |
|------------|---------|
| 简单 VS/PS（几十个绑定） | 数十微秒 |
| 大型 uber shader（数百绑定 + 深嵌套块） | 亚毫秒级 |

因为不解析函数体，耗时几乎只与**类型/装饰数量**相关，与指令总数无关。

**优化建议**：

```cpp
// 并行反射：每个线程独立模块
std::vector<ShaderReflectionData> baked(spvFiles.size());
std::for_each(std::execution::par, indices.begin(), indices.end(), [&](size_t i) {
    rt::ShaderModule m(ReadFileBinary(spvFiles[i]));
    baked[i] = Bake(m);       // 拷出 POD，模块在线程内销毁
});
```

---

## 14. 命令行工具

仓库的 `main.cpp` 编译出 `spirv-reflect`，是排查问题的利器：

```bash
# 完整 dump（绑定表、块布局、接口变量、入口点、能力集）
spirv-reflect shader.frag.spv

# 只看描述符绑定
spirv-reflect shader.frag.spv | grep -A5 "Descriptor binding"

# 配合反汇编交叉验证
spirv-dis shader.frag.spv > a.txt
spirv-reflect shader.frag.spv > b.txt
```

**输出结构**（大致）：

```
Shader stage:          FRAGMENT
Source language:       HLSL
Entry point:           PSMain
Generator:             Khronos Glslang / DXC ...
Shader capabilities:   Shader ...
Descriptor bindings:
  Binding 0:
    SPIR-V id:  12
    Name:       PerFrame
    Set:        0
    Binding:    0
    Type:       VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER
    Accessed:   true
    Decoration: BLOCK
    Block:
      Size:     80
      Member 0: viewProj    offset 0   size 64
      Member 1: cameraPos   offset 64  size 16
  ...
```

> **最佳实践**：把 `spirv-reflect` 的输出作为**构建产物的伴生文件**（`.spv.txt`）一起提交或生成，code review 时可以直接 diff 出"这次改动是否意外改变了描述符布局"。

---

## 15. 坑清单（按发生频率排序）

| # | 现象 | 根因 | 对策 |
|---|------|------|------|
| 1 | 描述符不匹配 / 画面黑 | 改写绑定后仍用**旧 SPIR-V** 建 `VkShaderModule` | 改写后一律 `spvReflectGetCode()` |
| 2 | `name` 为空 | 跑了 `spirv-opt -O`（strip debug） | 反射逻辑不依赖名字；或构建时保留 NonSemantic |
| 3 | `count == 0` 导致 binding 无效 | 运行时长度数组（bindless） | `count==0` → 用引擎容量兜底 + `PARTIALLY_BOUND` |
| 4 | `set` 号溢出报错 | set ≥ 64 | 约束 shader 的 `space` 编号 |
| 5 | `EntryPoint*` 返回 0 条 | SPIR-V < 1.4 | 回退到模块级枚举 + `accessed` 过滤 |
| 6 | 布局莫名多一个 binding | HLSL `RWStructuredBuffer` 的 counter buffer | 处理 `uav_counter_binding` |
| 7 | UBO 数据错位 | glslang/DXC 矩阵主序不同 | 永远以反射的 `RowMajor/ColMajor` 为准 |
| 8 | `size` 参数传错 | 传了字数而非字节数 | `words * sizeof(uint32_t)` |
| 9 | PushConstant 过大 | 超过 `maxPushConstantsSize`（通常 128B） | 反射后校验 + 合并区间 |
| 10 | 顶点属性多出内建量 | 没过滤 `BuiltIn` | 检查 `built_in` / `DECORATION_BUILT_IN` |
| 11 | `COMBINED_IMAGE_SAMPLER` 写入不全 | 只写了 image 没写 sampler | 一个 `VkDescriptorImageInfo` 同时填两者 |
| 12 | 悬垂指针 | 缓存了模块内的 `const char*` | 拷成 `std::string` |
| 13 | 数组维度截断 | 超过 32 维 | 规范上限本来就是 32，改 shader |
| 14 | 解析失败但信息不足 | 非法 SPIR-V | 反射前先 `spirv-val` |

### 15.1 CI 静态断言（防止库升级破坏假设）

```cpp
// 保证 SPIRV-Reflect 的枚举与 Vulkan 头保持对齐
static_assert(SPV_REFLECT_DESCRIPTOR_TYPE_SAMPLER
              == static_cast<int>(VK_DESCRIPTOR_TYPE_SAMPLER));
static_assert(SPV_REFLECT_DESCRIPTOR_TYPE_UNIFORM_BUFFER
              == static_cast<int>(VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER));
static_assert(SPV_REFLECT_DESCRIPTOR_TYPE_ACCELERATION_STRUCTURE_KHR
              == static_cast<int>(VK_DESCRIPTOR_TYPE_ACCELERATION_STRUCTURE_KHR));
static_assert(SPV_REFLECT_SHADER_STAGE_FRAGMENT_BIT
              == static_cast<int>(VK_SHADER_STAGE_FRAGMENT_BIT));
```

---

## 16. 小结

| 主题 | 要点 |
|------|------|
| 集成 | 单文件 C，零依赖；三种方式任选 |
| API 模式 | `Enumerate` 一律"两遍调用"；`size` 参数是字节数 |
| 描述符 | `set/binding/descriptor_type/count`；`count==0` 是 bindless 信号 |
| 布局 | `block.members[].absolute_offset` 是唯一真相 |
| 入口点 | 多入口点用 `module.entry_points[i]`；过滤依赖 SPIR-V ≥1.4 |
| 改写 | `Change*` + `GetCode`；**改写后必须用新 SPIR-V** |
| 生命周期 | 模块销毁后所有 `name`/指针失效；推荐烘成自己的 POD |
| 边界 | 名字依赖调试信息；语义（PerFrame 等）SPIR-V 里没有 |

---

*上一篇：[01-SPIR-V 模块结构与反射原理](01-SPIR-V模块结构与反射原理.md) | 下一篇：[03-反射方案对比与选型](03-反射方案对比与选型.md)*
