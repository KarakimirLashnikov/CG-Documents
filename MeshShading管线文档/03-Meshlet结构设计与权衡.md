# 03-Meshlet结构设计与权衡：顶点复用、簇级剔除与尺寸

> 本文档是「Mesh Shading 管线文档」系列的第 3 篇。把 meshlet 当作一个**数据契约**来分析：它带来什么、牺牲什么、V 与 T 该怎么定。
> 上一篇：[02-使用流程与传统管线迁移](02-使用流程与传统管线迁移.md)

---

## 1. Meshlet 是硬件约束的数据侧投影

一个 meshlet 必须满足三条硬性约束：

```
┌──────────── Meshlet（= 一个 Mesh Shader 工作组的工作量）────────────┐
│                                                                     │
│  vertices[0..V)      本簇用到的顶点（去重后）                        │
│  triangles[0..T)     每个三角形 3 个索引，索引值 ∈ [0, V)  ←─────────┼── ① 局部性
│                                                                     │
│  V ≤ maxMeshOutputVertices      ←── ② 顶点上限（设备相关，常见 256） │
│  T ≤ maxMeshOutputPrimitives    ←── ③ 图元上限（设备相关，常见 256） │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
        约束 ① 的代价：跨簇边界的顶点必须复制
        → 顶点冗余（duplication factor > 1）
```

**所有 meshlet 的设计权衡，本质上都是约束 ①②③ 与三个收益之间的三角关系。**

---

## 2. 收益一：顶点复用

### 2.1 从"硬件猜"到"离线定"

| 阶段 | 谁决定顶点被变换几次 | 是否可预测 |
|------|-------------------|-----------|
| 原始扫描网格 | 硬件切 primitive subgroup | 否，跨厂商/跨代不一致 |
| `optimizeVertexCache` 后 | 硬件切 subgroup，但顺序已被"讨好" | 近似可预测 |
| **静态 meshlet** | **离线切分完全决定** | **是，与硬件无关** |

AMD 的 Stanford Bunny 实测（约 34.8k 顶点 / 约 70k 三角形）：

| 数据状态 | 被变换顶点数 | Duplication Factor |
|---------|------------|-------------------|
| 原始 | 143.9k | 4.13 |
| 顶点缓存优化后 | 48.9k | 1.40 |
| **静态 meshlet** | **43.2k** | **1.24** |

解读：meshlet 的复用率**优于**顶点缓存优化，而且是**确定**的。这是 meshlet 最扎实、最无争议的收益。

### 2.2 冗余从哪来：周长 / 面积

```
一个 meshlet 是曲面上的一块区域：
   内部顶点：被本簇多个三角形共享 → 复用好
   边界顶点：还要被邻簇复制一份 → 冗余

   redundancy ≈ 边界周长 / 区域面积

   → 簇越大，周长/面积比越小 → 冗余越低
   → 簇越小，边界占比越高     → 冗余越高
```

这就是"**簇越大，顶点复用越好**"的几何直觉，也是 AMD 推荐 128/256 而不是 64/126 的核心理由。
⚠️ 但簇大还有另外两个代价（§3 占用率、§4 剔除粒度），所以不能无限大。

---

## 3. 收益二：簇级剔除

### 3.1 剔除数据的构成

`meshopt_computeMeshletBounds` 为每个 meshlet 产出：

| 字段 | 用途 | 剔除方式 |
|------|------|---------|
| `center` + `radius` | 包围球 | 视锥剔除、遮挡剔除（投影成屏幕 AABB） |
| `cone_apex` + `cone_axis` + `cone_cutoff` | 法线锥 | **锥形剔除**：若整簇全背向相机则整簇丢弃 |

```glsl
// 锥形剔除（meshoptimizer README 给出的判据）
if (dot(normalize(cone_apex - camera_position), cone_axis) >= cone_cutoff)
    reject();
```

⚠️ 锥形剔除的前提是**簇内三角形朝向足够一致**。所以它和"簇的形状"直接耦合：
- 理想的簇：空间紧凑、法线方向集中 → 锥角小 → 剔除率高；
- 糟糕的簇：把不相连的区域硬凑成一个簇（meshoptimizer 默认行为会这么做）→ 锥角接近 180° → 锥形剔除完全失效。

### 3.2 三者的三角关系

```
                    ┌──────────────┐
                    │  簇尺寸 ↑     │
                    └──────┬───────┘
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
     顶点冗余 ↓        剔除粒度 ↑      占用率 ↓
     （好事）          （坏事：剔除      （坏事：输出区
                        不精确）         预留变大）
```

| 指标 | 簇变大 | 簇变小 |
|------|--------|--------|
| Duplication factor | ↓ 好 | ↑ 坏 |
| 剔除精度 | ↓ 坏（一个簇跨半个屏幕，很难整簇剔掉） | ↑ 好 |
| 剔除开销占比 | ↓ 好（每簇固定开销被摊薄） | ↑ 坏 |
| Shader export 占用 | ↑ 坏（每簇预留更多） | ↓ 好 |
| 工作组调度粒度 | ↑ 粗（负载不均风险） | ↓ 细（更均衡，但固定开销上升） |
| 锥角 | ↑ 大（锥形剔除失效） | ↓ 小（剔除更狠） |

### 3.3 cone_weight：把"形状"加进目标函数

`meshopt_buildMeshlets(..., cone_weight)` 就是用来在 **拓扑效率（顶点复用）** 与 **剔除效率（锥角、半径）** 之间调权重的：

| `cone_weight` | 行为 | 何时用 |
|---------------|------|--------|
| `0.0` | 完全不管锥角，只优化顶点复用 | **不用锥形剔除时必须为 0**（官方明确） |
| `0.25` | 官方给出的合理默认 | 用锥形剔除 + 视锥/遮挡剔除混合时 |
| `→1.0` | 越来越偏向"锥角小、半径小" | 剔除率比顶点复用更重要的场景 |

⚠️ 反直觉点：`cone_weight > 0` 会**降低**顶点复用（为了形状牺牲拓扑）。所以"用了 cone_weight 反而变慢"是可能的，要实测。

### 3.4 需要空间局部性时的选择

meshoptimizer 默认"尽量把簇填满，哪怕把不相连的区域合进一个簇"。这对光栅化（顶点复用优先）是对的，但对下面两种场景是错的：

| 场景 | 需求 | 用什么 |
|------|------|--------|
| 层级 LOD（Nanite 式） | 簇必须空间局部，才能建立父子关系 | `meshopt_buildMeshletsFlex(min_tri, max_tri, split_factor≈2.0)` |
| 光线追踪簇加速结构 | 最小化 ray-triangle 求交 → 要 SAH 空间划分 | `meshopt_buildMeshletsSpatial(max_verts, min_tri=16, max_tri=64, fill_weight=0.5)` |

⚠️ 用 Flex 时，`meshopt_buildMeshletsBound` 的第三个参数要传 **`min_triangles`**（不是 max），否则上界算错。

---

## 4. 尺寸权衡：V 与 T 怎么定

### 4.1 先记住一个规律：V < T < 2V

对近似 2-manifold 的簇：

```
V < T < 2V        （T 接近 V   → T 成为瓶颈，多数簇先撞三角形上限）
                  （T 接近 2V  → V 成为瓶颈，多数簇先撞顶点上限）
```

| 配置 | 谁先撞上限 | 效果 |
|------|-----------|------|
| `T̂ = V̂`（如 64/64、128/128） | 三角形数 | 顶点被充分利用，但三角形少 → 簇数多 |
| `T̂ = 2V̂`（如 128/256） | 顶点数 | 每簇三角形多，顶点复用好 |

AMD 的建议：**让 V̂ 是线程组大小的整数倍，让 T̂ 不成为瓶颈**（因为顶点变换才是 Mesh Shader 的主要计算量）。

### 4.2 三套主流配置

| 配置 | 推荐方 | 线程组 | 理由 |
|------|--------|--------|------|
| **V=128, T=256** | AMD（GPUOpen 第 2 篇） | 128 | 边界/面积比更小 → 顶点冗余更低（bunny 1.24）；线程组 128 = D3D12 上限 |
| **V=64, T=126** | NVIDIA（Kubisch，meshoptimizer 引用） | 64 | 首代硬件图元索引按 **128 字节粒度**分配，需预留 4 字节存图元数：`3×126+4 = 382 ≤ 384 = 3×128` |
| **V=64, T=96** | meshoptimizer 补充建议 | 64 | 真实网格上 126 很难填满；96 更现实，也减轻其他 GPU 的开销 |

NVIDIA 还给出另外两个"对齐友好"的上限：**84** 与 **40**。
meshoptimizer 提示：老 AMD GPU 上 **V 与 T 取相同值**（64/64 或 128/128）往往更好；v0.25 之前 `max_triangles` 必须是 4 的倍数（所以常见 124 而非 126）。

⚠️ **没有 universal 最优解**。定死的唯一原因是资产已经烘死——所以离线参数与 shader 的 `layout(max_vertices, max_primitives)` 必须**由同一份配置生成**，且 shader 声明要 ≤ 设备上限。

### 4.3 线程组大小的三条规则（AMD）

```
规则 1：线程组大小 = V̂（一线程一顶点）
        → 128/256 配 128 线程；64/126 配 64 线程

规则 2：做不到 1:1 时，至少让 V̂ 能被线程数整除

规则 3：V̂=64 时，只有当 T̂ < 128 才需要调小线程组；
        否则"多一个线程专门导出多余图元"通常更快
        （AMD：having a dedicated thread for exporting extra primitives is faster）
```

反例（要避免）：

```hlsl
// ❌ 128 个线程，但只有前 64 个线程干活，后 64 个空转
if (threadId < 64) { verts[threadId] = ...; }
// ✅ 用步长让全部线程都参与
for (uint i = 0; i < 2; ++i) {
    uint pi = threadId + i * 128;
    if (pi < primCount) triangles[pi] = ...;
}
```

AMD 驱动实现侧的解释（Timur Kristóf 的博客，二手但方向一致）：
- MS 工作组大小要匹配最大顶点/图元数，并让工作均匀分布到整个工作组；
- 尽量让线程只写"自己那个下标"的输出数组元素 —— 否则驱动要把输出写进 LDS 再读回来，走的是慢路径；
- 用 shared memory 可以，但**优先用 subgroup 操作**替代。

### 4.4 分配粒度：声明 1 个顶点可能占 32 个

```
meshOutputPerVertexGranularity    ：顶点分配粒度（示例值 32）
meshOutputPerPrimitiveGranularity ：图元分配粒度（示例值 32）

→ 声明 max_vertices = 1，实际可能按 32 预留
→ 声明 64 与 65，占用可能一样；但 33 与 32 也一样
```

⚠️ 推论：**V/T 最好取粒度（常见 32）的整数倍**。取 126 这种值，图元侧会向上取整到 128，正好也是对齐的；但取 100 这种值就是纯粹浪费。

---

## 5. 数据布局

### 5.1 描述符式布局（meshoptimizer 推荐形态）

```c
struct Meshlet {
    uint32_t vertex_offset;    // 在 meshlet_vertices[] 中的偏移
    uint32_t vertex_count;
    uint32_t triangle_offset;  // 在 meshlet_triangles[] 中的字节/元素偏移
    uint32_t triangle_count;
    // 运行时再加：bounds（center/radius/cone），通常单独一个数组（SoA 更缓存友好）
};
// 伴随数组：
//   meshlet_vertices[] : uint32（或 uint16）→ 全局顶点缓冲索引
//   meshlet_triangles[]: uint8_t × 3 × triCount，按 4 字节对齐
```

```glsl
layout(binding = 0) readonly buffer Meshlets        { uvec4 meshlets[];        };
layout(binding = 1) readonly buffer MeshletVertices { uint   meshlet_vertices[]; };
layout(binding = 2) readonly buffer MeshletTriangles{ uint8_t meshlet_triangles[]; };

void main() {
    uvec4 m = meshlets[gl_WorkGroupID.x];
    SetMeshOutputsEXT(m.y, m.w);
    for (uint i = gl_LocalInvocationIndex; i < m.y; i += gl_WorkgroupSize.x) { ... }
    ...
}
```

⚠️ `uint8_t` 数组在 GLSL 里是合法但**读取粒度受限**的类型（一次只能读 1 字节，无法合并）。AMD 的建议是"把每个三角形重打包成 32 位以便对齐读取"。详见第 02 篇 §3.2 与第 06 篇 §2。

### 5.2 布局决策表

| 决策 | 选项 | 取舍 |
|------|------|------|
| meshlet 元数据 | AoS（一个 struct）vs SoA（多个数组） | SoA：剔除时只读 bounds，带宽小 |
| meshlet_vertices 类型 | `uint32` vs `uint16` | `uint16` 省一半；顶点数 > 65535 的巨型网格用 `uint32` |
| meshlet_triangles 类型 | `uint8` vs 打包成 `uint32` | `uint8` 省内存但逐字节读；打包 32 位便于合并访存 |
| 是否生成迷你 VB | 见第 02 篇 §3.1 | 省一次 gather，代价是内存膨胀 |
| 裁剪尾部 | `resize(last.offset + last.count)` | **必须做**：`buildMeshlets` 的数组是按最坏情况超额分配的 |

### 5.3 每 meshlet 后处理

```c
// ① 裁剪（必做，否则上传的是按最坏情况分配的大数组）
const meshopt_Meshlet& last = meshlets[meshlet_count - 1];
meshlet_vertices.resize(last.vertex_offset + last.vertex_count);
meshlet_triangles.resize(last.triangle_offset + ((last.triangle_count * 3 + 3) & ~3));
meshlets.resize(meshlet_count);

// ② 簇内再做一次局部优化（推荐）
meshopt_optimizeMeshlet(&meshlet_vertices[m.vertex_offset],
                        &meshlet_triangles[m.triangle_offset],
                        m.triangle_count, m.vertex_count);
```

`meshopt_optimizeMeshlet` 的作用是**在簇内**再排一次顶点与三角形顺序，提高簇内三角形/顶点局部性 —— 这直接影响 Mesh Shader 里的 gather 是否合并（第 06 篇）。

---

## 6. 什么时候不要 meshlet 化

| 场景 | 原因 | 建议 |
|------|------|------|
| 顶点数很少的网格（< ~1k 三角形） | 簇数太少，Task Shader 的固定开销占比高 | 走传统管线或一个大簇 |
| 顶点属性极宽（蒙皮、多 UV） | 变换成本高 → 冗余顶点的代价被放大 | **更应该** meshlet 化（复用收益更大），但要更认真地选 V |
| 每帧形变的动态几何 | 簇的拓扑与 bounds 会变 | 保留簇拓扑，只更新顶点；bounds 用保守包围盒 |
| 稀疏拓扑（粒子、线条） | 簇内复用天然低 | NVIDIA 建议：为稀疏拓扑单独写一套实现 |
| 只需画几百个物体的小场景 | CPU 开销本来就不构成瓶颈 | 别为架构纯度付出复杂度 |

---

## 7. 小结

| 要点 | 说明 |
|------|------|
| meshlet 本质 | 硬件三条约束（局部索引 / V 上限 / T 上限）在数据侧的投影 |
| 顶点复用 | 静态切分把复用率变成离线确定值，与硬件无关（bunny 4.13 → 1.24） |
| 冗余来源 | 边界周长 / 区域面积 → 簇越大冗余越低 |
| 簇级剔除 | 包围球（视锥/遮挡）+ 法线锥（背向）；锥角依赖簇形状，不是免费的 |
| 三角关系 | 簇变大：冗余↓、剔除精度↓、export 占用↑ |
| `cone_weight` | 0 = 不管锥角（不用锥形剔除时必须 0）；0.25 = 合理默认 |
| 尺寸规律 | `V < T < 2V`；V̂ 取线程组整数倍，T̂ 不成为瓶颈 |
| 三套配置 | AMD 128/256 + 128 线程；NVIDIA 64/126（382B 对齐）+ 64 线程；meshopt 建议试 64/96 |
| 分配粒度 | `meshOutputPer*Granularity`（常 32）→ V/T 尽量取其整数倍 |
| 线程组 | = V̂；做不到就整除；V̂=64 且 T̂≥128 时不必调小线程组 |
| 后处理 | 必做裁剪；推荐 `meshopt_optimizeMeshlet` |
| 别用场景 | 极小网格、稀疏拓扑、小场景 —— 复杂度换不来收益 |

---

*上一篇：[02-使用流程与传统管线迁移](02-使用流程与传统管线迁移.md) | 下一篇：[04-meshoptimizer设计理念与模块](04-meshoptimizer设计理念与模块.md)*
