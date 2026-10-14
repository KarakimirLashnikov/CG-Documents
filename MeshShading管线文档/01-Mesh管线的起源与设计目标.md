# 01-Mesh管线的起源与设计目标：它到底取代了什么

> 本文档是「Mesh Shading 管线文档」系列的第 1 篇。用硬件实现的证据说明传统几何管线的瓶颈，再落到 Vulkan `VK_EXT_mesh_shader` 的 API 事实清单。
> 上一篇：[00-总览](00-总览.md)

---

## 1. 传统几何管线的四个结构性瓶颈

### 1.1 瓶颈一：顶点复用不可控（由硬件猜，不受你控制）

带索引的绘制里，GPU 不能"先把所有顶点变换完再装配"——那需要存下可能几百万个变换后顶点，而且会阻塞图元装配。
所以硬件必须**边变换边装配**：取一批图元，收集这批用到的去重顶点索引，变换这批顶点，然后用"重索引后的局部索引"装配。

AMD 把这一步叫 **primitive subgroup**（RDNA NGG 流水线）：

```
索引缓冲: [4,5,6, 8,4,6, ...]
                 │
                 ▼  Geometry Engine 取一批三角形
┌────────────────────────────────────────────────┐
│ primitive subgroup（≈ 硬件版 meshlet）           │
│   unique vertex list : [4, 5, 6, 8]             │
│   primitive connectivity: (0,1,2) (3,0,2)       │
│   ↑ 局部索引，只能指向本 subgroup 内的顶点        │
└────────────────────────────────────────────────┘
                 │
                 ▼  Primitive Shader 逐线程变换 + exp 导出
┌────────────────────────────────────────────────┐
│ Shader Export（片上）→ Primitive Assembler → Raster │
└────────────────────────────────────────────────┘
```

关键推论：**顶点只在 subgroup 内部复用**。跨 subgroup 的边界顶点会被重复变换。
于是"到底变换了多少顶点"取决于硬件怎么切 subgroup —— 而这**对你不可见、不可控**，只能通过重排索引缓冲去"猜"（meshoptimizer 的 `meshopt_optimizeVertexCache` 做的就是这件事，第 04 篇）。

AMD 用 Stanford Bunny（约 34.8k 顶点 / 约 70k 三角形，扫描网格，顶点呈栅格排布）给了一组实测：

| 数据状态 | 被变换顶点数 | Duplication Factor |
|---------|------------|-------------------|
| 原始扫描网格 | 143.9k | 4.13 |
| `meshopt_optimizeVertexCache` 之后 | 48.9k | 1.40 |
| 静态切成 meshlet | 43.2k | **1.24** |

两个结论：

1. 顶点缓存优化是"猜硬件行为"，**跨厂商跨代不保证一致**；
2. 静态 meshlet 把复用率变成**离线确定**的，且与硬件无关。

### 1.2 瓶颈二：几何阶段的表达力被固定功能锁死

| 你想做的事 | 传统管线里的代价 |
|-----------|----------------|
| 压缩过的索引/顶点数据 | IA 只认固定格式（`VK_INDEX_TYPE_UINT16/32`、顶点属性格式），**无法**在装配前解压 |
| 程序化生成几何（草、发、体素等值面） | 只能靠实例 + 属性，或 CPU 生成，无法在 GPU 上按需要的粒度展开 |
| 整簇剔除 | 只能在 CPU 按物体做；GPU 上没有"一块几何"这个可寻址单位 |
| 按簇选 LOD | 只能整体换网格；per-object LOD 在密集场景里粒度太粗 |
| 自定义图元装配 / 细分 / 邻接查询 | 得走 GS，而 GS 历史上**跨厂商效率都很差** |

Khronos 的 `VK_EXT_mesh_shader` proposal 里对"为什么不改 VS 而另开两个阶段"给过明确论证（概括）：

- 方案 A：给 VS 加分组 + subgroup 通信 → 组内增删顶点/图元能力极弱，且受 subgroup 宽度限制；
- 方案 B：跳过 VS 直接用 GS/TS → 输入、输出拓扑、粒度仍然固定，且历史效率差；
- 方案 C（最终采纳）：**用两个 compute-like 阶段替代全部 pre-rasterization 阶段**，保留 compute 的全部灵活性，只重新定义"如何把结果喂给光栅器"。

### 1.3 瓶颈三：几何决策全在 CPU，帧开销随场景增长

物体级剔除 → 写 indirect 参数 → 按材质/网格分组发 draw call。
场景复杂度一上去，CPU 端是 `O(可见物体数 × 材质数)`，而且**每次 draw call 都要重新绑定顶点/索引缓冲、可能重绑描述符**。

Mesh Shading 把"可见集计算"整个搬进 GPU：CPU 只发一次 `vkCmdDrawMeshTasksEXT(wholeSceneGrid)`，剩下的由 Task Shader 展开。

### 1.4 瓶颈四：计算单元在几何阶段吃不饱

现代 GPU 是统一着色器（unified shader）架构：几何阶段跑在和 compute 一样的 CU/SM 上，但用的是"逐顶点"的编程模型——一个线程只看得到一个顶点，线程之间没有可见性、不能通信。
同一块硬件跑 compute 时能用的手段（workgroup、shared memory、subgroup 前缀和、barrier）在 VS 里一个都没有。

> **一句话**：Mesh Shading 的设计目标就是让几何阶段用上**和 compute 一样完整的工具箱**，同时保留"直接喂给光栅器"这条专线。

---

## 2. 时间线（含出处与时效）

| 时间 | 事件 | 备注 |
|------|------|------|
| 2018–2019 | NVIDIA Turing 引入 Mesh Shading，`VK_NV_mesh_shader` / D3D12 先行 | 厂商扩展，1D `taskCount/firstTask` |
| 2019（AMD 口径） | Mesh Shading 进入 DirectX® 12 | 见 GPUOpen 系列第 1 篇 |
| 2022-03-08 | `VK_EXT_mesh_shader` Revision 1 发布（Christoph Kubisch / Daniel Koch / Patrick Mours 等） | 跨厂商版本：Task/Mesh，三维网格派发 |
| RDNA™ 1 起 | AMD **NGG（Next Generation Geometry）** 落地 | primitive shader / primitive subgroup，是硬件侧的前置基础 |
| 2024-12 | Vulkan 1.4 发布 | **Mesh shader 未进核心**（以 1.4.341 refpage 为准，2026-10） |

⚠️ 网上流传的"Vulkan 1.3 就有了 EXT_mesh_shader / 1.4 提升为核心"说法与 refpage 不一致。判定以 Khronos refpage 的 "Version History" 段为准：该页至今只有 "Revision 1, 2022-03-08 Initial revision"，无提升记录。

---

## 3. 它如何适配现代 GPU 硬件架构

### 3.1 硬件侧的"meshlet 化"早已存在

Mesh Shading 不是凭空发明的 API，它把硬件**已经在偷偷做的事**暴露成可编程接口：

| 硬件概念 | 传统管线中的角色 | Mesh Shading 中的对应 |
|---------|----------------|---------------------|
| Geometry Engine（AMD）/ 图元分发器 | 切 primitive subgroup、启动 VS | 切分工作变成**离线的 meshlet 构建** |
| Primitive Shader（AMD NGG） | 逐线程变换 + `exp` 导出 | Mesh Shader（软件可见、可写任意索引） |
| Shader Export | 存变换后顶点 + 图元连接关系 | Mesh Shader 的输出区，按 `max_vertices/max_primitives` 预留 |
| Primitive Assembler | 读 export 装配三角形 + 剔除 + 视口变换 | 不变；`CullPrimitiveEXT` 在此被消费 |
| Dual Compute Unit / SM | 执行 VS | 执行 Task/Mesh，可用 shared memory、barrier、subgroup |

### 3.2 与"统一着色器 + wave"的执行模型对齐

- Mesh Shader **通常按 wave64 编译**（AMD，64 线程/wave），128 线程的工作组 = 2 个 wave。
- ⚠️ 使用 `WavePrefixSum` 一类的 wave 内扫描指令可能让编译器切到 **wave32**，wave 数翻倍 —— **占用率看起来变高，实际并行的工作组数没变**。这是 AMD 明确点出的陷阱。
- 派发即 compute 派发：`vkCmdDrawMeshTasksEXT(groupCountX, groupCountY, groupCountZ)`，与 `vkCmdDispatch` 同形。

### 3.3 输出区是"启动前预留"的硬约束

```
Mesh 工作组启动流程（AMD RDNA 口径）：

  Cmd Processor 收到 draw
        │
        ▼
  Geometry Engine 按【编译期声明的 max_vertices / max_primitives】
  在 Shader Export 预留空间          ← ⚠️ 不是按实际输出量
        │
        ▼
  启动 Mesh 工作组
        │
        ▼
  SetMeshOutputsEXT(v, p) 声明【实际】输出量（≤ 声明上限）
        │
        ▼
  逐线程写 verts[i] / triangles[i] → exp → PA → Raster
```

三个直接推论（第 06 篇展开）：

1. **声明上限必须尽量贴近真实值**：写 `max_vertices=256` 但实际只用 64，等于白扔 4 倍 export 空间，占用率直接掉。
2. `SetMeshOutputsEXT` 要**尽早调用**：AMD 明确建议，早调用能把几何引擎分配内存的延迟藏起来。
3. 剔除掉的图元**仍然占空间**：`CullPrimitiveEXT` 只让 PA 跳过，不释放 export。

### 3.4 RDNA2 与 RDNA3 的导出差异（为什么要"i 号线程写 i 号元素"）

- `exp` 指令的导出顺序 = 线程在组内的顺序：**第 i 个线程必须导出第 i 个顶点/图元**，否则装配会错。
- 若第 i 个线程没产生第 i 个元素，数据就得经 **group shared memory** 在线程间倒手 → 延迟 + 资源占用上升。
- RDNA™ 2：一个线程只能导出 1 个顶点 + 1 个图元；V/T 超过线程数时，**驱动插入隐藏线程**帮忙导出，数据走 shared memory。
- RDNA™ 3：`exp` 支持 wave 级偏移，第 i 个线程可以直接导出第 i 个和第 **i+128** 个（N 通常 = 128）。

所以 AMD 的写法建议是**线程组步长（thread-group-sided stride）**：

```hlsl
[numthreads(128, 1, 1)]              // 线程组 = V̂ = 128
[outputtopology("triangle")]
void MeshShader(in uint groupId : SV_GroupID,
                in uint threadId : SV_GroupThreadID,
                out vertices VertexAttribute verts[128],
                out indices uint3 triangles[256],      // T̂ = 256 = 2 × 128
                out primitives PrimitiveAttribute prims[256])
{
    Meshlet meshlet = meshlets[groupId];
    SetMeshOutputCounts(meshlet.vertexCount, meshlet.primitiveCount);   // ← 尽早

    for (uint i = 0; i < 2; ++i) {                     // ← 步长写入
        const uint primitiveId = threadId + i * 128;
        if (primitiveId < meshlet.primitiveCount) {
            triangles[primitiveId] = LoadPrimitive(meshlet, primitiveId);
            prims   [primitiveId] = LoadPrimitiveAttributes(meshlet, primitiveId);
        }
    }
    if (threadId < meshlet.vertexCount)
        verts[threadId] = LoadVertex(meshlet, threadId);  // ← 1:1 线程↔顶点
}
```

⚠️ 这套规则在 Vulkan 侧由 `VkPhysicalDeviceMeshShaderPropertiesEXT` 的四个布尔量显式暴露，应读取后按厂商偏好走：

| 属性 | 为 TRUE 时的含义 |
|------|-----------------|
| `prefersLocalInvocationVertexOutput` | 顶点输出按 `LocalInvocationIndex` 索引写最快 |
| `prefersLocalInvocationPrimitiveOutput` | 图元输出按 `LocalInvocationIndex` 索引写最快 |
| `prefersCompactVertexOutput` | 剔除后应**紧凑化**顶点数组（不留空洞） |
| `prefersCompactPrimitiveOutput` | 剔除后应紧凑化图元数组，否则用 `CullPrimitiveEXT` 更好 |

（示例值：某 NVIDIA 设备 `prefersCompactPrimitiveOutput = true`、其余三个 false；**务必运行时查询**。）

---

## 4. Vulkan 侧 API 事实清单

### 4.1 特性与属性

```c
// 设备创建时挂到 VkPhysicalDeviceFeatures2 / VkDeviceCreateInfo 的 pNext
typedef struct VkPhysicalDeviceMeshShaderFeaturesEXT {
    VkBool32 taskShader;                          // 支持 Task Shader
    VkBool32 meshShader;                          // 支持 Mesh Shader（总开关）
    VkBool32 multiviewMeshShader;                 // mesh + multiview
    VkBool32 primitiveFragmentShadingRateMeshShader; // 逐图元写 shading rate
    VkBool32 meshShaderQueries;                   // 新增的 query 类型
} VkPhysicalDeviceMeshShaderFeaturesEXT;
```

属性（`VkPhysicalDeviceMeshShaderPropertiesEXT`，节选，含义按 refpage）：

| 字段 | 含义 | 备注 |
|------|------|------|
| `maxMeshOutputVertices` | 单个工作组可发射的最大顶点数 | 常见 256（示例：某 NVIDIA 设备 256） |
| `maxMeshOutputPrimitives` | 单个工作组可发射的最大图元数 | 常见 256 |
| `meshOutputPerVertexGranularity` | 顶点分配的粒度（向上取整） | 示例：32 |
| `meshOutputPerPrimitiveGranularity` | 图元分配的粒度 | 示例：32 |
| `maxMeshOutputMemorySize` | Mesh 输出变量可用的总字节上限 | 示例：32768 |
| `maxMeshPayloadAndOutputMemorySize` | payload + 输出的合计上限 | 示例：48128 |
| `maxMeshSharedMemorySize` / `maxMeshPayloadAndSharedMemorySize` | SMEM / payload+SMEM 上限 | 示例：28672 |
| `maxMeshWorkGroupInvocations` | 工作组内总线程数上限 | 常见 128 |
| `maxTaskPayloadSize` | Task → Mesh payload 上限 | 示例 16384 |
| `maxMeshWorkGroupTotalCount` / `maxMeshWorkGroupCount[3]` | 可派发的工作组总数 / 各维上限 | 示例 4194304 / (4194304, 65535, 65535) |
| `maxPreferredTaskWorkGroupInvocations` / `maxPreferredMeshWorkGroupInvocations` | 厂商偏好的工作组线程数 | 示例：32 |
| `prefers*Output`（4 个 `VkBool32`） | 输出写法偏好，见 3.4 表 | 违反不报错，只掉性能 |

⚠️ `maxPreferred*WorkGroupInvocations` 与四个 `prefers*` 是**偏好而非硬限制**：违反不会产生校验错误，但会偏离厂商的最优路径。

### 4.2 管线创建

- 仍用 `vkCreateGraphicsPipelines`，**不是**新入口。
- `VkGraphicsPipelineCreateInfo::pVertexInputState` 与 `pInputAssemblyState` **置 NULL**（mesh 管线不使用）。
- 阶段用 `VK_SHADER_STAGE_TASK_BIT_EXT` / `VK_SHADER_STAGE_MESH_BIT_EXT`。
- **禁止**与 `VK_SHADER_STAGE_VERTEX_BIT` / `TESSELLATION_*` / `GEOMETRY_BIT` 混用。
- 必须有 Mesh Shader；Task Shader 可选。

```c
VkPipelineShaderStageCreateInfo stages[2] = {
    { .sType = ..., .stage = VK_SHADER_STAGE_TASK_BIT_EXT,
      .module = taskModule, .pName = "main" },
    { .sType = ..., .stage = VK_SHADER_STAGE_MESH_BIT_EXT,
      .module = meshModule, .pName = "main" },
};
VkGraphicsPipelineCreateInfo ci = {
    .pVertexInputState   = NULL,   // ⚠️ 必须 NULL
    .pInputAssemblyState = NULL,   // ⚠️ 必须 NULL
    .stageCount = 2, .pStages = stages,
    /* ... 其余（光栅化、颜色混合、渲染通路）与常规图形管线一致 ... */
};
```

### 4.3 派发命令

```c
// 直接派发：三维网格，与 vkCmdDispatch 同形
void vkCmdDrawMeshTasksEXT(VkCommandBuffer, uint32_t groupCountX,
                                            uint32_t groupCountY,
                                            uint32_t groupCountZ);

// 间接派发
void vkCmdDrawMeshTasksIndirectEXT(VkCommandBuffer, VkBuffer buffer,
                                   VkDeviceSize offset, uint32_t drawCount,
                                   uint32_t stride);

// 数量也来自 GPU
void vkCmdDrawMeshTasksIndirectCountEXT(VkCommandBuffer, VkBuffer buffer,
                                        VkDeviceSize offset, VkBuffer countBuffer,
                                        VkDeviceSize countBufferOffset,
                                        uint32_t maxDrawCount, uint32_t stride);

typedef struct VkDrawMeshTasksIndirectCommandEXT {
    uint32_t groupCountX, groupCountY, groupCountZ;
} VkDrawMeshTasksIndirectCommandEXT;
```

| 对比项 | `VK_EXT_mesh_shader` | `VK_NV_mesh_shader` |
|--------|---------------------|---------------------|
| 直接派发 | `(groupCountX, Y, Z)` 三维 | `(taskCount, firstTask)` 一维带偏移 |
| 无 Task 着色器时 | 直接派发 Mesh 工作组 | 同 |
| 推荐选择 | ✅ 跨厂商，用这个 | 仅 NVIDIA 早期/特殊需求 |

### 4.4 查询与同步

- 管线统计新增两位：`VK_QUERY_PIPELINE_STATISTIC_TASK_SHADER_INVOCATIONS_BIT_EXT`（0x800）、`VK_QUERY_PIPELINE_STATISTIC_MESH_SHADER_INVOCATIONS_BIT_EXT`（0x1000）。
- 独立查询类型：`VK_QUERY_TYPE_MESH_PRIMITIVES_GENERATED_EXT`（1000328000），统计**未被固定功能剔除**的、Mesh 阶段真正发射的图元数。
- 同步：`VK_PIPELINE_STAGE_TASK_SHADER_BIT_EXT` / `VK_PIPELINE_STAGE_MESH_SHADER_BIT_EXT`；`VK_PIPELINE_STAGE_2_PRE_RASTERIZATION_SHADERS_BIT` 已包含这两个阶段。

### 4.5 GLSL 骨架（与 HLSL/Slang 对照）

```glsl
#version 450
#extension GL_EXT_mesh_shader : require
#extension GL_EXT_buffer_reference : require   // 若用 BDA（见第 05 篇）

/* ─────────── Task Shader ─────────── */
layout(local_size_x = 32) in;              // 典型上限 128 invocations

struct TaskPayload { uint meshletOffset; uint count; float lodError; };
taskPayloadSharedEXT TaskPayload payload;   // SPIR-V: TaskPayloadWorkgroupEXT

void main() {
    /* 粗粒度剔除 / LOD 决策 */
    payload.meshletOffset = ...;
    payload.count         = visibleCount;
    EmitMeshTasksEXT(payload.count, 1, 1);  // 派发子工作组
}

/* ─────────── Mesh Shader ─────────── */
layout(local_size_x = 64) in;
layout(triangles) out;                      // 或 points / lines
layout(max_vertices = 64, max_primitives = 126) out;   // 典型上限 256/256

layout(location = 0) out vec4 vColor[];                 // 逐顶点（输出数组）
layout(location = 1) perprimitiveEXT out vec4 pNormal[]; // 逐图元（不插值）

void main() {
    SetMeshOutputsEXT(actualV, actualP);    // ⚠️ 必须在任何输出写之前
    gl_MeshVerticesEXT[i].gl_Position = ...;
    gl_PrimitiveTriangleIndicesEXT[j] = uvec3(a, b, c);
    gl_MeshPrimitivesEXT[j].gl_CullPrimitiveEXT = true;  // 逐图元剔除
}
```

内置块（write-only 数组）：

| GLSL | 内容 |
|------|------|
| `gl_MeshVerticesEXT[]` | `gl_Position` / `gl_PointSize` / `gl_ClipDistance[]` / `gl_CullDistance[]` |
| `gl_MeshPrimitivesEXT[]` | `gl_PrimitiveID` / `gl_Layer` / `gl_ViewportIndex` / `gl_CullPrimitiveEXT` / `gl_PrimitiveShadingRateEXT` |
| `gl_PrimitiveTriangleIndicesEXT[]` | `uvec3`（另有 Point 的 `uint` 与 Line 的 `uvec2` 版本） |

HLSL/Slang 对照：`SetMeshOutputsEXT` → `SetMeshOutputCounts`；`taskPayloadSharedEXT` → 出参 `out payload T`；`EmitMeshTasksEXT` → `DispatchMesh`。

---

## 5. 小结

| 要点 | 说明 |
|------|------|
| 起源 | 硬件（AMD NGG primitive subgroup / NVIDIA meshlet）早已在内部做"局部顶点复用"，Mesh Shading 把它暴露为可编程接口 |
| 问题一 | 顶点复用由硬件猜 → 静态 meshlet 把它变成离线确定（bunny 4.13 → 1.24） |
| 问题二 | IA/VS 表达力被固定功能锁死：不能解压、不能程序化、不能按簇决策 |
| 问题三 | 几何决策在 CPU，帧开销随场景增长 |
| 问题四 | 统一着色器上跑"逐顶点"模型，浪费了 workgroup/SMEM/subgroup 这套工具 |
| 设计取舍 | Khronos 否掉了"给 VS 加分组"和"用 GS 顶替"，选择两个 compute-like 阶段 |
| 硬件映射 | Mesh Shader ≈ AMD NGG primitive shader；输出区 = shader export，启动前预留 |
| 硬约束 | `max_vertices/max_primitives` 吃 export 空间；导出顺序 = 线程顺序 |
| Vulkan 事实 | `VK_EXT_mesh_shader` rev1（2022-03），**仍非核心（2026-10）**；三维派发；`pVertexInputState`/`pInputAssemblyState` 必须为 NULL |
| 反模式 | 用 Task/Mesh 去模拟固定功能管线 —— AMD 与 NVIDIA 官方均明确反对 |

---

*上一篇：[00-总览](00-总览.md) | 下一篇：[02-使用流程与传统管线迁移](02-使用流程与传统管线迁移.md)*
