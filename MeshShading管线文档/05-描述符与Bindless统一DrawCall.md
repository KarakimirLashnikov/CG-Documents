# 05-描述符与Bindless统一DrawCall：让 CPU 端只剩一次绑定

> 本文档是「Mesh Shading 管线文档」系列的第 5 篇。讲 Mesh Shader 里 Buffer/Image 描述符怎么用、Bindless 如何与 PSO / Shader / 描述符堆配合，把 CPU 端 DrawCall 压成"一个 PSO、一组绑定、一条 indirect 命令"。
> 上一篇：[04-meshoptimizer设计理念与模块](04-meshoptimizer设计理念与模块.md)

---

## 1. 为什么 Mesh Shading 天然要求 Bindless

传统管线里，"换个网格"= 换顶点/索引缓冲 + 换材质描述符。
Mesh 管线里没有 IA，几何数据是你自己从 SSBO/BDA 取的 —— 这意味着**"换网格"只是换一个整数下标**。

```
传统：  DrawCall = (PSO, VB, IB, 材质描述符, 实例数据, 变换)
        → 每换一个网格/材质就要 vkCmdBind* + vkCmdDrawIndexed

Mesh：  DrawCall = (PSO, [一个全局 bindless set], 一个 workgroup 网格)
        → 网格身份 = workgroup 里解出来的一个索引
        → 一个 draw 覆盖全场
```

⚠️ 但这是**可能性，不是必然性**。如果描述符还是"每网格一套 set"，你只是把 `vkCmdDrawIndexed` 换成了 `vkCmdDrawMeshTasksEXT`，draw call 数量一点没少。**Bindless 是让 Mesh Shading 真正省 CPU 的前提。**

---

## 2. 几何数据的载体：怎么选

| 载体 | 读取方式 | 大小限制 | 适用场景 | 备注 |
|------|---------|---------|---------|------|
| `VK_DESCRIPTOR_TYPE_STORAGE_BUFFER`（SSBO） | `readonly buffer` 运行时长度数组 | 受 `maxStorageBufferRange`（通常 ≥128 MB） | **Mesh Shader 几何数据的默认选择** | 可 `PARTIALLY_BOUND`、可做可变长度数组 |
| `VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER` | `uniform` block | 受 `maxUniformBufferRange`（常见 64 KB） | 每帧常量（相机、视锥） | ⚠️ 64 KB 对几何数据远远不够 |
| Buffer Device Address（BDA） | `GL_EXT_buffer_reference` / `buffer_reference` 指针 | 无描述符、无绑定点 | 指针化数据结构、GPU 自管理资源 | Vulkan 1.2 起核心（`VK_KHR_buffer_device_address`） |
| Push Constants | 着色器常量 | 规范保证 ≥128 字节 | 每 draw 的根数据（root index、batch base） | ⚠️ 极小，只够放索引 |
| Inline Uniform Block（IUB） | `VK_EXT_inline_uniform_block` | 通常 256–4 KB | 小量每 draw 数据，比 push constant 灵活 | 避免为几个 uint 创建描述符 |
| Image / Sampler | 纹理采样 | — | 材质贴图；VT/纹理流 | bindless 纹理数组的经典用法 |

**结论**：几何数据（meshlets / meshlet_vertices / meshlet_triangles / bounds / 全局 VB）用 **SSBO 或 BDA**；每帧常量用 **UBO**；每 draw 的根数据用 **push constant 或 IUB**；材质纹理用 **bindless 纹理数组**。

### 2.1 BDA vs SSBO 的取舍

| 维度 | SSBO（bindless 数组） | BDA |
|------|---------------------|-----|
| 绑定 | 需要描述符（但一次绑定全场） | **完全不需要描述符** |
| 索引方式 | `meshlets[i]`，i 是资源下标 | 直接是 64 位指针，可存在任何数据结构里 |
| 校验 | Validation layer 能查绑定 | **越界无校验**，写错就是垃圾数据或崩溃 |
| 对齐 | 由 std430 保证 | ⚠️ 必须自己保证指针对齐（常用 16 字节） |
| GPU 自生成资源 | 需要预先分配描述符 | **天然适合**：GPU 写入指针即可 |
| 可移植性 | 广泛 | 移动端支持面较窄（以设备查询为准） |

实践建议：**静态几何用 SSBO bindless 数组（可读性好、可校验）；流式/GPU 生成资源用 BDA。**

---

## 3. Bindless 三件套（Vulkan 侧配置清单）

### 3.1 ① `VK_EXT_descriptor_indexing`（Vulkan 1.2 起为核心特性）

需要启用的 feature（按需）：

```c
VkPhysicalDeviceDescriptorIndexingFeatures di = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_DESCRIPTOR_INDEXING_FEATURES,
    .shaderSampledImageArrayNonUniformIndexing      = VK_TRUE,  // 纹理非一致索引
    .shaderStorageBufferArrayNonUniformIndexing     = VK_TRUE,  // SSBO 非一致索引（Mesh 几何必需）
    .shaderUniformBufferArrayNonUniformIndexing     = VK_TRUE,
    .descriptorBindingSampledImageUpdateAfterBind   = VK_TRUE,
    .descriptorBindingStorageBufferUpdateAfterBind  = VK_TRUE,
    .descriptorBindingPartiallyBound                = VK_TRUE,  // ⚠️ 允许未初始化的槽位
    .descriptorBindingVariableDescriptorCount       = VK_TRUE,  // 运行时长度数组
    .descriptorBindingUpdateUnusedWhilePending      = VK_TRUE,  // 已提交的命令在用也能更新未用到的槽
    .runtimeDescriptorArray                         = VK_TRUE,  // 数组长度不必编译期已知
};
```

绑定标志（每个 binding 一个）：

| 标志 | 作用 | 何时必须 |
|------|------|---------|
| `VK_DESCRIPTOR_BINDING_PARTIALLY_BOUND_BIT` | 允许数组里有未写入的槽位 | **几乎总是**：流式加载时槽位会临时空缺 |
| `VK_DESCRIPTOR_BINDING_VARIABLE_DESCRIPTOR_COUNT_BIT` | 数组长度运行时决定 | 想按设备能力动态调整纹理数组大小时（⚠️ 规范对该标志的绑定位置有约束，通常放在编号最高的 binding） |
| `VK_DESCRIPTOR_BINDING_UPDATE_AFTER_BIND_BIT` | 描述符可在命令提交后更新 | 想边渲染边流式更新资源 |
| `VK_DESCRIPTOR_BINDING_UPDATE_UNUSED_WHILE_PENDING_BIT` | 即使 set 正在被使用，也能更新"未被引用"的槽 | 与上面配合，避免为每帧重建 set |

⚠️ `UPDATE_AFTER_BIND` 时，**描述符池也要** `VK_DESCRIPTOR_POOL_CREATE_UPDATE_AFTER_BIND_BIT`。

### 3.2 ② 非一致索引（NonUniform）

```glsl
#extension GL_EXT_nonuniform_qualifier : require

layout(set = 1, binding = 0) readonly buffer AllVertexBufs { float data[]; } vb[];

void main() {
    // ⚠️ 索引来自运行时数据 → 必须用 nonuniformEXT 包一层
    float x = vb[nonuniformEXT(mesh.bufferId)].data[i];
}
```

- 不加 `nonuniformEXT`：编译器可能做错误优化（把索引当一致值处理），在部分硬件上读到错误数据。
- 对应 feature：`shaderStorageBufferArrayNonUniformIndexing` / `shaderSampledImageArrayNonUniformIndexing`。

### 3.3 ③ 描述符堆的组织

```
┌──── 一个大的 descriptor pool（UPDATE_AFTER_BIND）────────────┐
│                                                              │
│  set 0（每帧 × 3）: 相机/视锥/全局参数 —— UBO + 可见簇列表     │
│  set 1（全局 × 1）: 几何 SSBO 数组（bindless, 可变长度）       │
│                    meshlets[] / meshlet_vertices[] / ...      │
│  set 2（全局 × 1）: 材质：bindless 纹理数组 + 材质 SSBO        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
   ↑ 整帧只绑定一次，之后所有 draw 复用
```

| set | 内容 | 更新频率 | 数量 |
|-----|------|---------|------|
| 0 | Frame UBO（相机、视锥平面、屏幕尺寸、时间）+ 可见簇/间接参数缓冲 | 每帧 | 3（帧重叠） |
| 1 | 几何：meshlets / meshlet_vertices / meshlet_triangles / bounds / 全局 VB | 加载/流式时 | 1（长期） |
| 2 | 材质：bindless 纹理数组（sampled image[]）+ 材质参数 SSBO + sampler 数组 | 流式时 | 1（长期） |

⚠️ 关键原则：**set 1、set 2 一旦绑定就不再重绑**。Mesh Shader 通过 `gl_WorkGroupID` / payload / push constant 里解出的整数去索引它们。

---

## 4. 代码骨架

### 4.1 描述符集布局

```c
// ── set 1：几何（bindless SSBO 数组）─────────────────────────────
VkDescriptorBindingFlagsEXT flags[4] = {
    VK_DESCRIPTOR_BINDING_PARTIALLY_BOUND_BIT | VK_DESCRIPTOR_BINDING_UPDATE_AFTER_BIND_BIT,  // meshlets
    VK_DESCRIPTOR_BINDING_PARTIALLY_BOUND_BIT | VK_DESCRIPTOR_BINDING_UPDATE_AFTER_BIND_BIT,  // meshlet_vertices
    VK_DESCRIPTOR_BINDING_PARTIALLY_BOUND_BIT | VK_DESCRIPTOR_BINDING_UPDATE_AFTER_BIND_BIT,  // meshlet_triangles
    VK_DESCRIPTOR_BINDING_PARTIALLY_BOUND_BIT | VK_DESCRIPTOR_BINDING_UPDATE_AFTER_BIND_BIT,  // bounds (SoA)
};
VkDescriptorSetLayoutBindingFlagsCreateInfoEXT flagInfo = {
    .sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_BINDING_FLAGS_CREATE_INFO_EXT,
    .bindingCount = 4, .pBindingFlags = flags,
};
VkDescriptorSetLayoutBinding b[4] = {
    { .binding = 0, .descriptorType = VK_DESCRIPTOR_TYPE_STORAGE_BUFFER,
      .descriptorCount = kMaxMeshes, .stageFlags = VK_SHADER_STAGE_TASK_BIT_EXT
                                                 | VK_SHADER_STAGE_MESH_BIT_EXT },
    /* ... 1,2,3 同理，bounds 还要给 TASK 阶段做剔除 ... */
};
VkDescriptorSetLayoutCreateInfo lci = { .pNext = &flagInfo, .bindingCount = 4, .pBindings = b };
```

```c
// ── pool 必须带 UPDATE_AFTER_BIND ────────────────────────────────
VkDescriptorPoolCreateInfo pci = {
    .flags = VK_DESCRIPTOR_POOL_CREATE_UPDATE_AFTER_BIND_BIT,   // ⚠️ 缺了这个会校验失败
    .maxSets = 8,
    .poolSizeCount = ..., .pPoolSizes = ...,
};
```

```c
// ── 更新：一次写全场，之后只增量写新槽 ────────────────────────────
VkDescriptorBufferInfo info = { .buffer = meshVertexBuffer, .offset = 0, .range = VK_WHOLE_SIZE };
VkWriteDescriptorSet w = {
    .sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET,
    .dstSet = geomSet, .dstBinding = 1, .dstArrayElement = meshIndex,   // ← 只写这一个槽
    .descriptorType = VK_DESCRIPTOR_TYPE_STORAGE_BUFFER, .descriptorCount = 1,
    .pBufferInfo = &info,
};
vkUpdateDescriptorSets(dev, 1, &w, 0, nullptr);
```

### 4.2 Shader 侧（GLSL）

```glsl
#version 450
#extension GL_EXT_mesh_shader            : require
#extension GL_EXT_nonuniform_qualifier   : require
#extension GL_EXT_scalar_block_layout    : require

// ── set 0：每帧 ─────────────────────────────────────────────
layout(set = 0, binding = 0) uniform Frame {
    mat4  viewProj;
    vec4  frustum[6];
    vec3  camPos;
    uint  totalMeshlets;
};

// ── set 1：几何（bindless，数组长度为运行时上限）─────────────
layout(set = 1, binding = 0) readonly buffer Meshlets  { uvec4  meshlets[];        } meshletBufs[];
layout(set = 1, binding = 1) readonly buffer MeshVerts { uint   mv[];              } meshVertBufs[];
layout(set = 1, binding = 2) readonly buffer MeshTris  { uint   mt[];              } meshTriBufs[];
layout(set = 1, binding = 3) readonly buffer Bounds    { vec4   sph[]; /*+ cone*/  } boundsBufs[];

// ── 每 draw 的根数据：push constant ─────────────────────────
layout(push_constant) uniform Root {
    uint meshId;        // → meshletBufs[meshId]
    uint firstCluster;  // 本物体在簇数组中的起点
    uint clusterCount;
    uint materialId;    // → 材质/纹理（bindless 索引）
};

layout(local_size_x = 64) in;
layout(triangles) out;
layout(max_vertices = 64, max_primitives = 126) out;

void main() {
    uint meshletId = firstCluster + gl_WorkGroupID.x;

    // ⚠️ nonuniformEXT：索引来自 push constant（运行时值）
    uvec4 m = meshletBufs[nonuniformEXT(meshId)].meshlets[meshletId];
    SetMeshOutputsEXT(m.y, m.w);

    uint tid = gl_LocalInvocationIndex;
    if (tid < m.y) {
        uint vi = meshVertBufs[nonuniformEXT(meshId)].mv[m.x + tid];
        gl_MeshVerticesEXT[tid].gl_Position = viewProj * LoadPosition(vi);
    }
    for (uint i = 0; i < 2; ++i) {
        uint pi = tid + i * 64;
        if (pi < m.w) {
            gl_PrimitiveTriangleIndicesEXT[pi] =
                DecodeTri(meshTriBufs[nonuniformEXT(meshId)].mt, m.z + pi);
        }
    }
}
```

⚠️ 上面把 `meshId` 放在 push constant 里 —— 这意味着**每个物体还是要一次 draw**。
要做到"全场一次 draw"，必须让 `meshId` 也来自 GPU：把"meshletId → meshId"的映射做成 **GPU 侧的查找表/可见簇列表**，由 Task Shader 或前置 compute 生成（见 §5）。

### 4.3 BDA 变体（去掉 set 1）

```glsl
#extension GL_EXT_buffer_reference : require
#extension GL_EXT_buffer_reference_uvec2 : require   // 需要时（指针存成 uvec2）

layout(buffer_reference, scalar) readonly buffer MeshletBuf { uvec4 v[]; };
layout(buffer_reference, scalar) readonly buffer VertBuf    { float v[]; };

layout(set = 0, binding = 1) readonly buffer ClusterTable {
    uvec2 meshletPtr;    // BDA（uvec2 形式，便于 GPU 侧写入）
    uvec2 vertPtr;
    uint  count;
} clusters[];            // ⚠️ 这个数组本身仍是 SSBO

void main() {
    MeshletBuf mb = MeshletBuf(clusters[cid].meshletPtr);   // BDA 还原成指针
    uvec4 m = mb.v[meshletId];
    ...
}
```

---

## 5. 统一 DrawCall：三种进阶层次

```
层次 0：每物体一次 direct draw
        vkCmdBindPipeline + bind + push(meshId) + vkCmdDrawMeshTasksEXT(n,1,1)
        → CPU 开销 ∝ 物体数；只是把 DrawIndexed 换了个名字

层次 1：每物体一条 indirect 记录（multi-draw）
        GPU 的剔除 pass 填好 VkDrawMeshTasksIndirectCommandEXT{x,y,z}
        CPU：vkCmdDrawMeshTasksIndirectEXT(buf, off, drawCount, stride)
        → CPU 开销 ∝ 批次数（仍需要 push constant 分批，或靠 stride 藏数据）

层次 2：全场一次 draw（真正的目标）
        GPU 生成“可见簇列表”，每个簇自带 (meshId, materialId, meshletId)
        CPU：vkCmdDrawMeshTasksIndirectCountEXT(...)  ← 连数量都来自 GPU
        → CPU 开销 = O(1)，与场景复杂度无关

层次 3：DGC（VK_EXT_device_generated_commands）
        命令流本身由 GPU 生成，token 支持 DRAW_MESH_TASKS_EXT / COUNT
        → CPU 连 command buffer 都不用录
```

### 5.1 Multi-draw 与 stride 的小技巧

```c
// VkDrawMeshTasksIndirectCommandEXT 只有 12 字节，但 stride 可以更大！
typedef struct VkDrawMeshTasksIndirectCommandEXT {
    uint32_t groupCountX, groupCountY, groupCountZ;
} VkDrawMeshTasksIndirectCommandEXT;

// 于是可以把“每 draw 的根数据”插在记录之间：
//   [cmd | root][cmd | root][cmd | root] ...
//   stride = sizeof(cmd) + sizeof(Root)
// 记录之间多出来的字节被跳过，shader 通过 gl_DrawIndex 自己算偏移去读。
uint32_t stride = sizeof(VkDrawMeshTasksIndirectCommandEXT) + sizeof(Root);
vkCmdDrawMeshTasksIndirectEXT(cmd, buf, 0, drawCount, stride);
```

⚠️ 这个技巧依赖 `gl_DrawIndex` 在 Mesh/Task 阶段的可用性（EXT_mesh_shader 把 `DrawIndex` 标为 modified built-in）。**实测确认**再用于生产。

### 5.2 与 PSO 的关系

| 决策 | 说明 |
|------|------|
| PSO 数量 | 理想是**一个**不透明几何 PSO 覆盖全场（材质差异走 bindless 索引 + 分支/数据驱动） |
| 为什么能这么少 | 网格差异变成整数索引，不再需要不同的 IA 布局（传统管线每个顶点格式就是一个 PSO 变体！） |
| 编译耗时 | PSO 少但 shader 可能很复杂 → 用 `VK_EXT_graphics_pipeline_library` 拆分编译，避免卡顿 |
| 动态状态 | 尽量把状态放进动态状态（视口/裁剪/深度/剔除模式），减少 PSO 变体 |
| Shader 变体 | ⚠️ 别为了"材质特性"复制 PSO；走 uber-shader + 分支，或者按**特性位**做有限几个变体 |

### 5.3 更新时序（Bindless 最容易出事的地方）

```
帧 N：
  ┌─ CPU: 流式加载新网格 → vkUpdateDescriptorSets(set1, slot=k)  ⚠️ 更新未使用的槽
  │       （需要 UPDATE_UNUSED_WHILE_PENDING，否则与正在执行的帧冲突）
  ├─ CPU: 录命令 → vkCmdBindDescriptorSets(set1 全局) → vkCmdDrawMeshTasks...
  └─ GPU: 执行
```

| 坑 | 现象 | 对策 |
|----|------|------|
| 更新"正在被 GPU 使用"的描述符 | 未定义行为 / 崩溃 / 画错 | 要么等 fence，要么用 `UPDATE_UNUSED_WHILE_PENDING` + 只更新确定未被本帧引用的槽 |
| 描述符池耗尽 | `vkAllocateDescriptorSets` 失败 | 预分配足够大的池；或按类型分池 |
| `PARTIALLY_BOUND` 的槽读到垃圾 | 偶发花屏/崩溃 | 流式加载完成前，**确保簇不会进入可见列表**；或用 null descriptor 兜底 |
| 忘记 `nonuniformEXT` | 部分硬件读错资源 | 所有运行时索引一律包 `nonuniformEXT` |
| SSBO 对齐 | 数据整体偏移、读到错值 | 用 `std430`（`scalar` 更好）；结构体手动对齐；BDA 指针 16 字节对齐 |
| 越界读 | 读到别的资源 | 打开 `robustBufferAccess2`（`VK_EXT_pipeline_robustness`）做**开发期**校验，发布期关掉 |

---

## 6. 完整的一帧（CPU 侧伪代码）

```c
// ── 初始化（一次）─────────────────────────────────────────────
createBindlessLayouts();          // set0 每帧 / set1 几何 / set2 材质
allocatePool(UPDATE_AFTER_BIND);
allocateSets();
writeAllGeometry();               // 一个网格 = 一个数组槽
createSingleMeshPSO();            // 一个 PSO 画全场不透明几何

// ── 每帧 ─────────────────────────────────────────────────────
vkBeginCommandBuffer(cmd);
  // 1) GPU 侧：剔除 pass（compute）→ 可见簇列表 + indirect 参数
  vkCmdBindPipeline(cmd, COMPUTE, cullPipeline);
  vkCmdDispatch(cmd, (totalClusters + 63) / 64, 1, 1);

  // 2) 屏障：compute write → task/mesh read
  vkCmdPipelineBarrier2(cmd, &(VkDependencyInfo){
      .pBufferMemoryBarriers = &(VkBufferMemoryBarrier2){
          .srcStageMask = VK_PIPELINE_STAGE_2_COMPUTE_SHADER_BIT,
          .srcAccessMask = VK_ACCESS_2_SHADER_WRITE_BIT,
          .dstStageMask = VK_PIPELINE_STAGE_2_TASK_SHADER_BIT_EXT
                        | VK_PIPELINE_STAGE_2_MESH_SHADER_BIT_EXT,
          .dstAccessMask = VK_ACCESS_2_SHADER_READ_BIT, ... }});

  // 3) 画：一个 PSO + 一组绑定 + 一条命令
  vkCmdBeginRendering(cmd, &renderingInfo);
  vkCmdBindPipeline(cmd, GRAPHICS, meshPipeline);
  vkCmdBindDescriptorSets(cmd, GRAPHICS, layout, 0, 3, allSets, 0, nullptr);
  vkCmdDrawMeshTasksIndirectCountEXT(cmd, indirectBuf, 0,
                                     countBuf, 0, kMaxBatches, stride);
  vkCmdEndRendering(cmd);
vkEndCommandBuffer(cmd);
```

CPU 每帧命令数：**常量级**，与场景中有多少物体、多少簇无关。

---

## 7. 必须纠正的说法

| 说法 | 事实 |
|------|------|
| "Mesh Shader 自动减少 draw call" | 不会。**Bindless + indirect 才减少**；不改描述符策略，draw 数一模一样 |
| "Bindless 就是少调 vkUpdateDescriptorSets" | 它是让"一个 PSO + 一次绑定 + 一条 indirect draw"成立的前提 |
| "SSBO 数组随便索引" | 运行时索引必须 `nonuniformEXT`，并启用 `shaderStorageBufferArrayNonUniformIndexing` |
| "PARTIALLY_BOUND 很方便，开了就行" | 代价是**读到未初始化槽位**不报错。必须保证可见列表里没有未就绪的簇 |
| "BDA 更好，全部用 BDA" | BDA 无校验、无绑定检查、对齐自负；静态几何用 SSBO 更安全、可调试 |
| "PSO 会变多（每材质一个）" | 恰恰相反：Mesh 管线**不需要顶点输入布局**，PSO 变体数通常比传统管线**更少** |
| "push constant 能塞下每 draw 数据" | 规范保证只有 128 字节；大数据走 IUB 或 GPU 侧的表 |
| "indirect 命令结构只有 12 字节，没法带参数" | stride 可以大于结构体大小，多余字节可用来穿插每 draw 数据（需实测 `gl_DrawIndex` 语义） |

---

## 8. 小结

| 要点 | 说明 |
|------|------|
| 为什么必须 Bindless | 没有 IA 后"换网格"只是换一个整数；不换描述符模型则 draw call 数不变 |
| 几何载体 | SSBO（默认、可校验）/ BDA（无描述符、适合 GPU 自生成）/ UBO 只放每帧常量 |
| 三件套 | `VK_EXT_descriptor_indexing` 的 flags + `nonuniformEXT` + 大 pool（UPDATE_AFTER_BIND） |
| 布局 | set0 每帧 ×3；set1 几何（长期一次绑定）；set2 材质纹理数组 |
| 目标形态 | 单 PSO + 一次 `vkCmdBindDescriptorSets` + 一条 `vkCmdDrawMeshTasksIndirectCountEXT` |
| 进阶 | stride > 12 可在 indirect 记录间穿插根数据；DGC 连命令流都由 GPU 生成 |
| PSO 影响 | 不再需要顶点输入布局 → PSO 变体数下降；用 `VK_EXT_graphics_pipeline_library` 分摊编译 |
| 四个高危点 | 更新已提交 set 的时序、PARTIALLY_BOUND 的空槽、缺 `nonuniformEXT`、SSBO/BDA 对齐 |
| 调试建议 | 开发期开 `robustBufferAccess2` + GPU 断言（`meshletId < total` 早退），发布期关掉 |

---

*上一篇：[04-meshoptimizer设计理念与模块](04-meshoptimizer设计理念与模块.md) | 下一篇：[06-性能优化与常见隐患](06-性能优化与常见隐患.md)*
