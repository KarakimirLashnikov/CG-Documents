# 实时渲染系统 — Vulkan 管线与着色器系统

> 本文档是「实时渲染系统」系列文档的第 **2** 篇，详细阐述 Vulkan 四类管线类型、各管线对应的着色器阶段、PipelineLayout 与描述符系统、以及 Bindless 无绑定扩展的落地方案。

---

## 1. Vulkan 管线类型体系

Vulkan 定义了四类管线绑定类型，每种管线拥有独立的着色器阶段集合与资源绑定规则：

```
┌───────────────────────────────────────────────────────────┐
│                  Vulkan Pipeline Types                     │
│                                                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│  │  图形管线    │  │  计算管线    │  │  光线追踪    │       │
│  │  GRAPHICS   │  │  COMPUTE    │  │  RAY_TRACING │       │
│  └─────────────┘  └─────────────┘  └─────────────┘       │
│                                                           │
│  ┌─────────────┐                                          │
│  │  Mesh管线   │  (图形管线扩展子集)                        │
│  │  (Task+Mesh)│                                          │
│  └─────────────┘                                          │
└───────────────────────────────────────────────────────────┘
```

### 管线绑定点枚举

| Vulkan 常量 | 值 | 说明 |
|-------------|---|------|
| `VK_PIPELINE_BIND_POINT_GRAPHICS` | 0 | 传统图形渲染 + Mesh Shader 渲染 |
| `VK_PIPELINE_BIND_POINT_COMPUTE` | 1 | 计算着色器任务 |
| `VK_PIPELINE_BIND_POINT_RAY_TRACING_KHR` | 1000165000 | 光线追踪渲染 |

---

## 2. 图形管线详解

### 2.1 传统图形管线着色器阶段

传统 Vulkan 图形管线包含五个可选着色器阶段，按执行顺序排列：

```
输入装配 ──▶ 顶点着色器 ──▶ 细分控制 ──▶ 细分评估 ──▶ 几何着色器 ──▶ 片段着色器 ──▶ 输出合并
   (IA)        (VS)        (TCS)       (TES)        (GS)         (FS)       (OM)
```

| 阶段 | Vulkan 枚举 | 必选/可选 | 核心职责 |
|------|-------------|---------|---------|
| 顶点着色器 | `VK_SHADER_STAGE_VERTEX_BIT` | **必选** | 逐顶点变换：模型空间→裁剪空间 |
| 细分控制着色器 | `VK_SHADER_STAGE_TESSELLATION_CONTROL_BIT` | 可选 | 设定细分级别、传递细分 Patch 数据 |
| 细分评估着色器 | `VK_SHADER_STAGE_TESSELLATION_EVALUATION_BIT` | 可选 | 生成细分后顶点位置、属性插值 |
| 几何着色器 | `VK_SHADER_STAGE_GEOMETRY_BIT` | 可选 | 逐图元增删顶点（输出点/线/三角带） |
| 片段着色器 | `VK_SHADER_STAGE_FRAGMENT_BIT` | **必选** | 逐像素着色：材质参数输出、光照计算 |

**实际使用建议**：

- **细分着色器（TCS/TES）**：用于曲面细分（地形、角色面部），在现代引擎中逐步被 Mesh Shader 替代
- **几何着色器（GS）**：性能较差，现代引擎极少使用，仅在特殊需求（如视锥体可视化）时启用
- **顶点+片段**：最常用的最小图形管线组合

### 2.2 Mesh Shader 图形管线（VK_EXT_mesh_shader）

Mesh Shader 是图形管线的**新一代替代方案**，取代传统 VS→GS 流程：

```
输入装配 ──▶ Task着色器 ──▶ Mesh着色器 ──▶ 片段着色器 ──▶ 输出合并
   (IA)       (Task)        (Mesh)        (FS)         (OM)
```

| 阶段 | Vulkan 枚举 | 必选/可选 | 核心职责 |
|------|-------------|---------|---------|
| Task Shader | `VK_SHADER_STAGE_TASK_BIT_EXT` | 可选 | Meshlet 集群级剔除，决定哪些 Mesh Shader 组需要执行 |
| Mesh Shader | `VK_SHADER_STAGE_MESH_BIT_EXT` | **必选** | 直接输出顶点与图元（无需传统顶点装配），替代 VS+GS |
| Fragment Shader | `VK_SHADER_STAGE_FRAGMENT_BIT` | **必选** | 与传统片段着色器一致 |

**Mesh Shader 核心优势**：

1. **集群级剔除**：Task Shader 在 Meshlet 粒度执行剔除，比逐顶点剔除更高效
2. **灵活输出**：Mesh Shader 可动态决定输出顶点/图元数量（无需预知绘制命令）
3. **GPU 自主调度**：配合 GPU-Driven Pipeline 实现全 GPU 渲染流程
4. **与传统管线不兼容**：Mesh Shader 管线不可包含 VS/TCS/TES/GS

### 2.3 图形管线状态配置

创建图形管线需配置大量固定功能状态：

```cpp
VkGraphicsPipelineCreateInfo pipelineInfo = {};
pipelineInfo.sType = VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO;

// 必需状态
pipelineInfo.pVertexInputState   = &vertexInputInfo;    // 顶点输入绑定+属性
pipelineInfo.pInputAssemblyState = &inputAssembly;      // 拓扑模式(三角列表/带)
pipelineInfo.pViewportState      = &viewportState;      // 视口+裁剪矩形
pipelineInfo.pRasterizationState = &rasterizer;         // 填充模式+背面剔除+深度偏移
pipelineInfo.pMultisampleState   = &multisampling;      // MSAA采样数+Alpha-to-Coverage
pipelineInfo.pDepthStencilState  = &depthStencil;       // 深度测试+写入+ stencil操作
pipelineInfo.pColorBlendState    = &colorBlend;         // 混合模式+逻辑操作
pipelineInfo.pDynamicState       = &dynamicState;       // 运行时可动态修改的状态

// 着色器阶段
pipelineInfo.pStages = shaderStages.data();
pipelineInfo.stageCount = shaderStages.size();

// 管线布局（描述符+Push Constant）
pipelineInfo.layout = pipelineLayout;

// 渲染Pass（子Pass描述）
pipelineInfo.renderPass = renderPass;
pipelineInfo.subpass = 0;
```

**动态状态优化**：

```cpp
// 指定可动态修改的状态，避免为每种组合创建独立管线
VkPipelineDynamicStateCreateInfo dynamicState = {};
VkDynamicState dynamicStates[] = {
    VK_DYNAMIC_STATE_VIEWPORT,
    VK_DYNAMIC_STATE_SCISSOR,
    VK_DYNAMIC_STATE_LINE_WIDTH,
    VK_DYNAMIC_STATE_DEPTH_BIAS,
    VK_DYNAMIC_STATE_BLEND_CONSTANTS,
    VK_DYNAMIC_STATE_DEPTH_BOUNDS,
    VK_DYNAMIC_STATE_STENCIL_COMPARE_MASK,
    VK_DYNAMIC_STATE_STENCIL_REFERENCE,
};
dynamicState.pDynamicStates = dynamicStates;
dynamicState.dynamicStateCount = 9;
```

> **建议**：尽量使用动态状态而非创建多个管线变体，配合 `VK_EXT_extended_dynamic_state` 可覆盖更多状态。

---

## 3. 计算管线详解

### 3.1 计算管线着色器阶段

计算管线仅有**一个着色器阶段**：

```
计算着色器 (Compute Shader) → 独立调度
```

| 阶段 | Vulkan 枚举 | 核心职责 |
|------|-------------|---------|
| Compute Shader | `VK_SHADER_STAGE_COMPUTE_BIT` | 通用 GPU 计算：剔除、模拟、滤波、降采样等 |

### 3.2 计算管线创建

计算管线的创建比图形管线简单得多：

```cpp
VkComputePipelineCreateInfo computeInfo = {};
computeInfo.sType = VK_STRUCTURE_TYPE_COMPUTE_PIPELINE_CREATE_INFO;
computeInfo.stage.sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
computeInfo.stage.stage = VK_SHADER_STAGE_COMPUTE_BIT;
computeInfo.stage.module = computeShaderModule;
computeInfo.stage.pName = "main";
computeInfo.layout = pipelineLayout;  // 与图形管线共享布局体系

VkPipeline computePipeline;
vkCreateComputePipelines(device, pipelineCache, 1, &computeInfo, nullptr, &computePipeline);
```

### 3.3 计算着色器调度模型

Compute Shader 以**工作组（Workgroup）**为单位调度：

```glsl
// GLSL Compute Shader
layout(local_size_x = 256, local_size_y = 1, local_size_z = 1) in;

void main() {
    uint globalID = gl_GlobalInvocationID.x;
    // 逐元素处理
}
```

```cpp
// Vulkan 调度命令
// dispatch(groupsX, groupsY, groupsZ)
vkCmdDispatch(cmdBuffer,
    ceil(objectCount / 256.0f),  // X维度工作组数
    1,                            // Y维度工作组数
    1);                           // Z维度工作组数
```

**工作组维度选择原则**：

| 场景 | local_size_x | 说明 |
|------|-------------|------|
| 逐物体剔除 | 64~256 | 每线程处理一个物体 |
| 逐像素图像处理 | 8×8×1 (2D) | 2D 工作组匹配图像块 |
| 逐顶点蒙皮 | 128~256 | 每线程处理一个顶点 |
| Hi-Z 降采样 | 8×8×1 | 匹配 2×2 降采样块 |

### 3.4 计算管线在渲染系统中的核心角色

计算管线承担了**预处理阶段**和**后处理阶段**的大部分工作：

| Pass | 计算管线角色 | 调度规模 |
|------|-------------|---------|
| GPU 剔除 | 逐物体剔除计算 | N物体 / 256 组 |
| Hi-Z 生成 | 深度降采样 | 图像尺寸 / 8 组 |
| 蒙皮变换 | 逐顶点矩阵变换 | N顶点 / 256 组 |
| SSR Ray March | 逐像素反射搜索 | 图像尺寸 / 8 组 |
| TAA 混合 | 逐像素时域混合 | 图像尺寸 / 8 组 |
| Bloom 降采样 | 逐级图像降采样 | 图像尺寸 / 8 组 |
| 超分推理 | 逐像素 AI 超分 | 图像尺寸 / 8 组 |

---

## 4. 光线追踪管线详解

### 4.1 光线追踪着色器阶段

光线追踪管线包含六大着色器阶段，按射线生命周期排列：

```
Raygen ──▶ Intersection ──▶ Any-Hit ──▶ Closest-Hit ──▶ Miss
   │            │              │           │           │
   │            │              │           │           ▼
   │            │              │           │      Shader输出
   │            │              │           ▼
   │            │              │      ──▶ Callable (可调用辅助)
   │            │
   ▼        自定义几何
 射线发射     交叉测试
```

| 阶段 | Vulkan 枸举 | 核心职责 |
|------|-------------|---------|
| **Raygen** | `VK_SHADER_STAGE_RAYGEN_BIT_KHR` | 射线生成入口，从像素/光源/反射面发射射线 |
| **Intersection** | `VK_SHADER_STAGE_INTERSECTION_BIT_KHR` | 自定义几何交叉测试（非三角几何如 AABB/SDF） |
| **Any-Hit** | `VK_SHADER_STAGE_ANY_HIT_BIT_KHR` | 射线沿途任意交叉点回调（透明物体过滤） |
| **Closest-Hit** | `VK_SHADER_STAGE_CLOSEST_HIT_BIT_KHR` | 射线最近交叉点着色计算（光照/阴影/反射） |
| **Miss** | `VK_SHADER_STAGE_MISS_BIT_KHR` | 射线未命中任何几何时的处理（天空/背景色） |
| **Callable** | `VK_SHADER_STAGE_CALLABLE_BIT_KHR` | 可调用的辅助函数（随机数、纹理采样、LOD选择） |

### 4.2 加速结构（Acceleration Structure）

光线追踪管线依赖**加速结构（AS）**实现高效的射线-几何交叉测试：

```
┌─────────────────────────────────────────┐
│          顶层加速结构 (TLAS)              │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐       │
│  │Obj A│ │Obj B│ │Obj C│ │Obj D│       │
│  └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘       │
│     ▼       ▼       ▼       ▼           │
│   BLAS     BLAS     BLAS     BLAS       │
│  (底层)   (底层)   (底层)   (底层)       │
└─────────────────────────────────────────┘

TLAS: 场景级实例树，支持动态实例变换矩阵更新
BLAS: 单物体级几何BVH，构建后通常不变
```

**Vulkan 加速结构创建**：

```cpp
// BLAS 创建（底层加速结构，每个网格一个）
VkAccelerationStructureGeometryBLASDataKHR blasData = {};
blasData.geometryType = VK_GEOMETRY_TYPE_TRIANGLES_KHR;
blasData.vertexFormat = VK_FORMAT_R32G32B32_SFLOAT;
blasData.vertexData.deviceAddress = vertexBufferAddress;
blasData.vertexStride = sizeof(Vertex);
blasData.maxVertexCount = vertexCount;
blasData.indexType = VK_INDEX_TYPE_UINT32;
blasData.indexData.deviceAddress = indexBufferAddress;
blasData.indexCount = indexCount;

// TLAS 创建（顶层加速结构，场景级）
VkAccelerationStructureInstanceKHR instance = {};
instance.instanceCustomIndex = materialIndex;   // Bindless材质索引
instance.accelerationStructureReference = blasAddress;
instance.transform = transformMatrix3x4;
instance.flags = VK_GEOMETRY_INSTANCE_FORCE_OPAQUE_BIT_KHR;
instance.mask = 0xFF;
instance.shaderBindingTableRecordOffset = 0;

// 动态更新：仅重建 TLAS（BLAS 可不变）
vkCmdBuildAccelerationStructuresKHR(cmdBuffer, 1, &tlasBuildInfo, &tlasBuildRangeInfo);
```

### 4.3 Shader Binding Table（SBT）

光线追踪管线通过 **Shader Binding Table** 将射线类型映射到对应着色器：

```
SBT 内存布局:
┌────────────┬────────────┬────────────┬────────────┬────────────┬────────────┐
│  Raygen    │  Miss      │  Hit       │  Callable  │            │            │
│  Region    │  Region    │  Region    │  Region    │            │            │
└────────────┴────────────┴────────────┴────────────┴────────────┴────────────┘

每个 Region 按 shaderBindingTableRecordOffset 索引:
  offset = rayTypeIndex * stride + baseOffset
```

```cpp
VkStridedDeviceAddressRegionKHR raygenRegion = {};
raygenRegion.deviceAddress = raygenBufferAddress;
raygenRegion.stride = sizeof(RaygenRecord);
raygenRegion.size = raygenCount * sizeof(RaygenRecord);

VkStridedDeviceAddressRegionKHR missRegion = {};
missRegion.deviceAddress = missBufferAddress;
missRegion.stride = sizeof(MissRecord);
missRegion.size = missCount * sizeof(MissRecord);

VkStridedDeviceAddressRegionKHR hitRegion = {};
hitRegion.deviceAddress = hitBufferAddress;
hitRegion.stride = sizeof(HitRecord);
hitRegion.size = hitCount * sizeof(HitRecord);

vkCmdTraceRaysKHR(cmdBuffer,
    &raygenRegion, &missRegion, &hitRegion, &callableRegion,
    width, height, 1);  // 每像素发射一条射线
```

---

## 5. 管线布局与描述符系统

### 5.1 PipelineLayout 架构

每个 Vulkan 管线都绑定到一个 `VkPipelineLayout`，它定义了管线的**资源访问接口**：

```cpp
VkPipelineLayoutCreateInfo layoutInfo = {};
layoutInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_PIPELINE_LAYOUT_CREATE_INFO;

// 描述符集布局（0~N组，每组包含多个描述符绑定）
layoutInfo.pSetLayouts = descriptorSetLayouts.data();
layoutInfo.setLayoutCount = descriptorSetLayouts.size();

// Push Constant 范围（快速小数据通道）
layoutInfo.pPushConstantRanges = pushConstantRanges.data();
layoutInfo.pushConstantRangeCount = pushConstantRanges.size();

VkPipelineLayout pipelineLayout;
vkCreatePipelineLayout(device, &layoutInfo, nullptr, &pipelineLayout);
```

### 5.2 描述符集层次结构

```
PipelineLayout
├─ 描述符集 0 (全局Bindless)       ← 跨管线共享，每帧/永久更新
│  ├─ Binding 0: 全局纹理数组       ← CombinedImageSampler[4096]
│  ├─ Binding 1: 全局Buffer数组     ← StorageBuffer[4096]
│  └─ Binding 2: 全局加速结构数组    ← AccelerationStructureKHR[256]
│
├─ 描述符集 1 (帧级Uniform)        ← 每帧更新
│  ├─ Binding 0: 帧Uniform Buffer   ← 相机、光照参数、时间
│  ├─ Binding 1: ShadowMap纹理集    ← 各光源阴影纹理
│
├─ 描述符集 2 (Pass级资源)         ← 每Pass更新
│  ├─ Binding 0: Pass输入附件       ← GBuffer/深度/运动矢量
│  ├─ Binding 1: Pass输出Buffer     ← 间接命令/剔除结果
│
└─ Push Constant Range             ← 快速逐物体/逐Pass参数
   ├─ Offset 0: 变换矩阵           ← 128 bytes (VK_MAX_PUSH_CONSTANTS_SIZE_MIN)
   ├─ Offset 128: 材质参数索引      ← 4 bytes
```

**设计原则**：

- **Set 0（全局Bindless）**：全管线共享，低频更新 → 使用 `VK_DESCRIPTOR_BINDING_UPDATE_AFTER_BIND_BIT`
- **Set 1（帧级）**：全帧共享，每帧更新一次 → 使用 Uniform Buffer Dynamic
- **Set 2（Pass级）**：每 Pass 更新 → 描述符池每帧重置分配
- **Push Constant**：极高频逐调用数据 → 无描述符开销，直接推送

### 5.3 描述符类型详解

| 描述符类型 | Vulkan 枚举 | 典型用途 | Update策略 |
|-----------|-------------|---------|-----------|
| Uniform Buffer | `VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER` | 帧级/Pass级参数 | Dynamic UBO 偏移切换 |
| Storage Buffer | `VK_DESCRIPTOR_TYPE_STORAGE_BUFFER` | GPU读写数据 | Bindless 数组索引 |
| Combined Image Sampler | `VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER` | 纹理采样 | Bindless 数组索引 |
| Storage Image | `VK_DESCRIPTOR_TYPE_STORAGE_IMAGE` | GPU写入纹理 | Pass级分配 |
| Acceleration Structure | `VK_DESCRIPTOR_TYPE_ACCELERATION_STRUCTURE_KHR` | 光追加速结构 | Bindless 数组索引 |
| Sampler | `VK_DESCRIPTOR_TYPE_SAMPLER` | 独立采样器 | 全局共享 |

---

## 6. Bindless 无绑定扩展

### 6.1 设计动机

传统 Vulkan 渲染中，每次绘制前需要逐绑定更新描述符，这造成：

- **CPU 开销**：逐物体/逐材质绑定切换
- **描述符池压力**：大量临时描述符分配/释放
- **管线兼容性约束**：不同绑定组合需兼容的管线布局

Bindless 方案通过**全局资源数组**消除逐绑定切换：

```
传统模式 (Per-Draw 绑定):
  Draw(obj0) → bind texture[0] → bind sampler → bind UBO → draw
  Draw(obj1) → bind texture[1] → bind sampler → bind UBO → draw
  Draw(obj2) → bind texture[2] → bind sampler → bind UBO → draw
  ... N次绑定切换

Bindless模式 (全局数组 + 索引):
  DrawAll() → bind global_array[ALL] → push material_index → draw all
  Shader: textureArray[material_index]  // 直接索引访问
```

### 6.2 Vulkan Bindless 落地

**核心扩展**：`VK_EXT_descriptor_indexing`

```cpp
// 描述符集布局标志：允许运行时更新、部分绑定
VkDescriptorSetLayoutBindingFlagsCreateInfo bindingFlags = {};
VkDescriptorBindingFlags flags[] = {
    VK_DESCRIPTOR_BINDING_PARTIALLY_BOUND_BIT |
    VK_DESCRIPTOR_BINDING_UPDATE_AFTER_BIND_BIT |
    VK_DESCRIPTOR_BINDING_VARIABLE_DESCRIPTOR_COUNT_BIT,
};
bindingFlags.pBindingFlags = flags;
bindingFlags.bindingCount = 1;

// 全局纹理数组绑定
VkDescriptorSetLayoutBinding textureArrayBinding = {};
textureArrayBinding.binding = 0;
textureArrayBinding.descriptorType = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
textureArrayBinding.descriptorCount = MAX_TEXTURE_COUNT;  // 4096+
textureArrayBinding.stageFlags = VK_SHADER_STAGE_ALL;
textureArrayBinding.pImmutableSamplers = nullptr;

// 创建 Bindless 描述符集
VkDescriptorSetLayoutBindless layoutBindless = {};
// ... 配置绑定与标志

VkDescriptorPool poolBindless = {};
// poolSize.descriptorCount = MAX_TEXTURE_COUNT;  // 大容量池

// 分配描述符集（带变量描述符计数）
VkDescriptorSetVariableDescriptorCountAllocateInfo varCountInfo = {};
uint32_t maxTexCount = currentTextureCount;  // 当前实际纹理数量
varCountInfo.pDescriptorCounts = &maxTexCount;
varCountInfo.descriptorSetCount = 1;

VkDescriptorSet bindlessSet;
vkAllocateDescriptorSets(device, &allocInfo, &bindlessSet);
```

### 6.3 Shader 侧 Bindless 使用

```glsl
// 全局纹理数组（Bindless）
layout(set = 0, binding = 0) uniform texture2D globalTextures[];
layout(set = 0, binding = 1) uniform sampler globalSamplers[];

// 全局 Buffer 数组（Bindless）
layout(set = 0, binding = 2) buffer StorageBuffer {
    vec4 data[];
} globalBuffers[];

// 全局加速结构数组（Bindless 光追）
layout(set = 0, binding = 3) accelerationStructureEXT globalAS[];

// 使用方式：通过材质索引直接访问
struct MaterialData {
    uint albedoTextureIndex;
    uint normalTextureIndex;
    uint ormTextureIndex;
    uint bufferIndex;
};

layout(push_constant) uniform PushConstants {
    uint materialIndex;
};

void main() {
    MaterialData mat = globalBuffers[mat.bufferIndex].data[materialIndex];
    vec3 albedo = texture(sampler2D(globalTextures[mat.albedoTextureIndex],
                                     globalSamplers[0]), uv).rgb;
}
```

### 6.4 Bindless 资源索引管理

全局索引分配器维护资源索引的分配与回收：

```cpp
class BindlessIndexAllocator {
    std::vector<uint32_t> freeIndices;     // 可用索引池
    uint32_t nextIndex = 0;                // 下一个新索引
    uint32_t maxIndex = MAX_BINDLESS_COUNT; // 上限

public:
    uint32_t Allocate() {
        if (!freeIndices.empty()) {
            uint32_t idx = freeIndices.back();
            freeIndices.pop_back();
            return idx;
        }
        assert(nextIndex < maxIndex);
        return nextIndex++;
    }

    void Free(uint32_t index) {
        // 延迟回收：标记为待释放，GPU帧完成后才真正回收
        pendingFree.push_back({index, currentFrameIndex});
    }

    void ProcessDeferredFree(uint64_t completedFrame) {
        for (auto& [idx, frame] : pendingFree) {
            if (frame <= completedFrame) {
                freeIndices.push_back(idx);
            }
        }
    }
};
```

---

## 7. 管线缓存与创建优化

### 7.1 PipelineCache 持久化

管线创建是 Vulkan 中最昂贵的操作之一。通过 PipelineCache 可大幅加速：

```cpp
// 创建管线缓存
VkPipelineCacheCreateInfo cacheInfo = {};
cacheInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_CACHE_CREATE_INFO;
VkPipelineCache pipelineCache;
vkCreatePipelineCache(device, &cacheInfo, nullptr, &pipelineCache);

// 使用缓存创建管线
vkCreateGraphicsPipeline(device, pipelineCache, 1, &pipelineInfo, nullptr, &pipeline);

// 持久化缓存到磁盘
size_t cacheSize;
vkGetPipelineCacheData(device, pipelineCache, &cacheSize, nullptr);
std::vector<char> cacheData(cacheSize);
vkGetPipelineCacheData(device, pipelineCache, &cacheSize, cacheData.data());
// 写入文件: pipeline_cache.bin
```

### 7.2 管线创建策略

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| **预编译全部变体** | 离线编译所有 Shader 变体组合 | 管线数量可控的中小项目 |
| **运行时按需创建** | 首次使用时创建，缓存后续复用 | 大型项目，变体数量爆炸 |
| **管线库** | `VK_EXT_pipeline_library`，模块化创建 | 大型引擎，支持并行创建 |
| **快速链接管线** | `VK_EXT_graphics_pipeline_library`，从库链接完整管线 | 减少创建开销 |

### 7.3 Shader 变体管理

Shader 变体是管线创建的核心挑战：

```
基础Shader: PBR.vert + PBR.frag
├─ 变体: #define ENABLE_NORMAL_MAP     → 管线 A
├─ 变体: #define ENABLE_EMISSION       → 管线 B
├─ 变体: #define ENABLE_NORMAL_MAP + ENABLE_EMISSION → 管线 C
├─ 变体: #define ALPHA_MASK            → 管线 D
├─ ... 组合爆炸: 2^N 种管线
```

**管理方案**：

- **离线编译**：Asset Pipeline 预编译所有已知变体 → PipelineCache 持久化
- **SPIR-V 反射**：自动提取描述符绑定、Push Constant 布局 → 自动生成管线布局代码
- **动态变体**：运行时根据材质属性选择变体 → PipelineCache 查找/创建
- **变体收集**：运行时记录实际使用的变体 → 下次离线预编译

---

## 8. 色器与管线对应关系总结

| 管线类型 | 色器阶段组合 | 最小必选 | 扩展可选 |
|---------|-------------|---------|---------|
| **传统图形** | VS + (TCS+TES) + (GS) + FS | VS, FS | TCS/TES, GS |
| **Mesh图形** | (Task) + Mesh + FS | Mesh, FS | Task |
| **计算** | Compute | Compute | — |
| **光线追踪** | Raygen + (Intersection) + (Any-Hit) + Closest-Hit + Miss + (Callable) | Raygen, Closest-Hit, Miss | Intersection, Any-Hit, Callable |

> **重要原则**：Mesh Shader 管线与传统顶点/细分/几何着色器**不兼容**，不可在同一管线中混合使用。

---

*上一篇：[01-渲染管线分层架构](01-渲染管线分层架构.md)*
*下一篇：[03-渲染架构选型与对比](03-渲染架构选型与对比.md)*
