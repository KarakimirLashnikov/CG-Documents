# 04-meshoptimizer设计理念与关键模块：离线一次，把"不可控"变成"可测"

> 本文档是「Mesh Shading 管线文档」系列的第 3+1 篇（第 4 篇）。讲 meshoptimizer 为什么这样设计、每个模块解决什么、以及每个取舍背后的原因。
> 上一篇：[03-Meshlet结构设计与权衡](03-Meshlet结构设计与权衡.md)

---

## 1. 设计理念

meshoptimizer（Arseny Kapoulkine / zeux）不是"网格库"，它是一条**离线几何处理流水线**。它的设计取向可以概括成五条：

| 理念 | 具体表现 | 为什么 |
|------|---------|--------|
| **① 一切在离线做一次** | 所有算法都是一次性预处理；运行时不重排、不重建 | 运行时预算应该留给渲染 |
| **② 用"近似模型"替代"实测"** | 提供 `meshopt_analyze*` 系列，用 ACMR/ATVR/overfetch/overdraw 描述质量 | 真机实测成本高、不可回归；近似指标可进 CI |
| **③ 顺序敏感，模块可组合** | 有明确的推荐调用顺序，后一步吃前一步的输出 | 每个算法都有"输入假设"，违反假设收益归零 |
| **④ 零依赖、可嵌入** | 单头文件式 C/C++ 实现，可换 allocator，栈占用 ≤ 32 KB | 要能塞进任意构建/资源管线 |
| **⑤ 无损为默认，有损显式化** | 只有 quantize/simplify 是有损；encode 是无损的 | 让"画质损失"这件事只发生在你点头的地方 |

⚠️ 官方明确声明：**分析器用的是 GPU 单元的近似模型，数字只是粗略近似**。它们的作用是"回归对比"和"发现异常"，不能替代真机测量。

---

## 2. 推荐流水线（顺序不可乱）

```
   源网格（可能无索引 / 有重复顶点）
        │
        ▼  ① meshopt_generateVertexRemap        → 去重（二进制等价）
        │  + meshopt_remapIndexBuffer / remapVertexBuffer
        ▼
   干净网格 ─────────────────────────────────────┐
        │                                        │
        │  ② meshopt_simplify × N   （LOD 链）    │  ← 每个 LOD 单独走一次 ③④
        │     记录 lod_error                     │
        ▼                                        │
   LOD0 │ ③ meshopt_optimizeVertexCache          │
        │     （桌面 GPU：post-transform cache）   │
        │     （移动端 FIFO：…CacheFifo(…, 16)）   │
        │                                        │
        │ ④ meshopt_optimizeOverdraw             │
        │     threshold 默认 1.05                │
        ▼                                        │
   ⑤ meshopt_optimizeVertexFetchRemap + remap    │
      + meshopt_optimizeVertexFetch              │
        │   （把顶点顺序变成索引顺序 → 顺序读）    │
        ▼                                        │
   ⑥ meshopt_quantize* （有损：法线八面体、UV 12bit…）
        │
        ▼  ⑦ meshopt_encodeVertexBuffer / encodeIndexBuffer  （无损压缩）
        │     encodeVertexVersion(1) / encodeVertexBufferLevel
        ▼
   ⑧ meshopt_buildMeshlets（+ optimizeMeshlet + computeMeshletBounds）
        │
        ▼
   GPU 上传
```

⚠️ 三条硬性顺序约束：

| 约束 | 原因 |
|------|------|
| `buildMeshlets` 必须在 `optimizeVertexCache` **之后** | meshlet 生成器要吃一个拓扑良好的索引顺序，否则簇内复用差 |
| `optimizeVertexFetch` 必须在 `buildMeshlets` **之前** | 它重排顶点 → 会让 meshlet 里存的顶点索引全部失效 |
| `encode*` 必须**最后** | 压缩依赖"已量化 + 已 vertex-fetch 优化"的数据；喂未优化数据进去压缩比会很差 |

---

## 3. 模块详解与取舍

### 3.1 顶点重映射（去重）

```c
std::vector<unsigned int> remap(vertex_count);
size_t vertex_count2 = meshopt_generateVertexRemap(
        &remap[0], indices ? &indices[0] : NULL, index_count,
        &vertices[0].px, unindexed_vertex_count, sizeof(Vertex));
meshopt_remapIndexBuffer(&indices[0], NULL, index_count, &remap[0]);
meshopt_remapVertexBuffer(&vertices[0], &unindexed[0],
                          unindexed_vertex_count, sizeof(Vertex), &remap[0]);
```

| 设计点 | 取舍原因 |
|--------|---------|
| 判定依据是**二进制等价**（逐字节） | 默认行为合理、快；但浮点漂移会产生"本该合并却没合并"的顶点 |
| ⚠️ **padding 必须清零** | 二进制比较包含 padding 字节；结构体有空洞却不清零 → 随机多出顶点 |
| 提供 `meshopt_generateVertexRemapCustom` | 允许自定义比较函数（带容差），解决浮点漂移 |

### 3.2 顶点缓存优化（顺序重排，不改变数据）

```c
meshopt_optimizeVertexCache(&indices[0], &indices[0], index_count, vertex_count);
// 移动端 FIFO 缓存架构：
meshopt_optimizeVertexCacheFifo(&indices[0], &indices[0], index_count, vertex_count, 16);
```

| 指标 | 定义 | 最好 / 最坏 |
|------|------|------------|
| **ACMR** | 顶点调用总数 ÷ 三角形数 | 规则网格 ≈0.5 / 最坏 3.0；真实网格 0.5–1.5 |
| **ATVR** | 顶点着色器调用数 ÷ 总顶点数 | 理论最优 1.0（每顶点只变换一次） |

取舍原因：
- 算法本质是 **tipsify 家族**（局部性优先的三角形重排），目标是让"共享顶点"的三角形在时间上靠近；
- ⚠️ 它优化的是**硬件 post-transform cache 的近似模型** —— 这意味着它**跨厂商、跨代不保证最优**。这正是第 01 篇说的"传统管线的顶点复用不可控"。

### 3.3 Overdraw 优化

```c
meshopt_optimizeOverdraw(&indices[0], &indices[0], index_count,
                         &vertices[0].px, vertex_count, sizeof(Vertex),
                         /* threshold = */ 1.05f);
```

| 设计点 | 说明 |
|--------|------|
| 目标 | 按"视角簇"重排三角形，让被遮挡的部分尽量晚画（更容易被 early-z 干掉） |
| `threshold = 1.05` | 允许的 ACMR 劣化上限：用 ≤5% 的顶点缓存损失换 overdraw 收益 |
| ⚠️ 副作用 | **会劣化顶点缓存**。所以必须放在 `optimizeVertexCache` **之后**，且要复测 ACMR |
| 何时跳过 | 有深度 prepass / 深度写入顺序本身就是最优时，收益有限 |

### 3.4 Vertex Fetch 优化（内存布局）

```c
std::vector<unsigned int> remap(vertex_count);
size_t vc = meshopt_optimizeVertexFetchRemap(&remap[0], &indices[0],
                                             index_count, vertex_count);
meshopt_remapVertexBuffer(&vertices[0], &vertices[0], vertex_count,
                          sizeof(Vertex), &remap[0]);
meshopt_remapIndexBuffer(&indices[0], &indices[0], index_count, &remap[0]);
```

- 指标：**overfetch** = 从 VB 读到的字节数 ÷ VB 总字节数，最优 1.0。
- 作用：把顶点按**索引中首次出现的顺序**重排，让一次绘制的顶点读取近似顺序 → 缓存行利用率上升。
- ⚠️ 它改变顶点顺序，所以**必须在 meshlet 化之前**做（见 §2）。

### 3.5 简化（LOD）

```c
float lod_error = 0.f;
lod.resize(meshopt_simplify(&lod[0], &indices[0], index_count,
                            &vertices[0].px, vertex_count, sizeof(Vertex),
                            target_index_count, target_error,
                            /* options = */ 0, &lod_error));
```

| 函数 | 特点 | 何时用 |
|------|------|--------|
| `meshopt_simplify` | 保持原拓扑；不跨越属性不连续处；约 3M 三角形/秒（桌面 CPU 口径） | 默认 |
| `meshopt_simplifySloppy` | 不保持拓扑/缝/边界，可合并空间邻近但拓扑分离的特征；**快 5–6 倍**，约 20M 三角形/秒 | 大比例简化、可接受接缝变化的场合 |
| `meshopt_simplifyWithAttributes` | 带属性权重（法线/UV/顶点色）参与误差 | 有顶点色、UV 变形敏感时 |
| `meshopt_simplifyScale` | 把归一化误差换算到物体空间 | 结合屏幕空间误差选 LOD |

`options` 位掩码（含义以 README/头文件为准）：

| 选项 | 含义 | 用途 |
|------|------|------|
| `meshopt_SimplifyLockBorder` | 锁定网格边界顶点，禁止塌缩 | **分块地形 / 分块 LOD 防裂** |
| `meshopt_SimplifySparse` | 输入是稀疏子集时加速；⚠️ 误差变为相对子集尺寸 | **逐 meshlet 简化**（建议同时用 `ErrorAbsolute`） |
| `meshopt_SimplifyErrorAbsolute` | 误差改为绝对值 | 配合 Sparse，或替代 `simplifyScale` |
| `meshopt_SimplifyPrune` | 允许移除孤立连通块 | 全网格简化时推荐（可能连带移除含锁定顶点的块） |

取舍原因：
- **误差是"相对网格包围盒尺寸"归一化的**（1e-2 ≈ 网格尺度的 1%），因为这样才能跨资产比较、才能和屏幕空间误差对接；
- 是否"保持拓扑"是最重要的取舍：保持拓扑才能安全复用原顶点缓冲、才能保持 UV 缝；不保持则能压得更狠 —— 所以两个算法都提供，让你按资产类型选。

### 3.6 量化与编码（有损 → 无损）

```c
// 有损：量化（把浮点压成小整数）
meshopt_quantizeHalf(&v.nx, ...)          // 法线/切线常用 half 或八面体
unsigned short q = meshopt_quantizeUnorm(x, 12);   // UV 用 12 bit（非 8 的倍数也行）
meshopt_quantizeFloat(&v.px, ..., 15);    // 单精度去尾数位熵

// 无损：编码
std::vector<unsigned char> vbuf(meshopt_encodeVertexBufferBound(vertex_count, sizeof(Vertex)));
vbuf.resize(meshopt_encodeVertexBuffer(&vbuf[0], vbuf.size(),
                                       &vertices[0], vertex_count, sizeof(Vertex)));
// 建议：meshopt_encodeVertexVersion(1)  → 更小更快，且支持 level 1/2/3
//       meshopt_encodeVertexBufferLevel(...) 可指定压缩等级

std::vector<unsigned char> ibuf(meshopt_encodeIndexBufferBound(index_count, vertex_count));
ibuf.resize(meshopt_encodeIndexBuffer(&ibuf[0], ibuf.size(), &indices[0], index_count));
```

| 事实 | 数值（官方口径） |
|------|------|
| 顶点压缩比 | 约 **2–4×**（相对"已量化且已打包"的数据） |
| 索引压缩比 | 约 **5–6×**（相对原始 16-bit 索引）；索引约 **1 字节/索引**，再叠通用压缩可到 1–3 bit/索引 |
| 解码速度 | 桌面 CPU 约 **3–6 GB/s**（版本不同口径略有差异） |
| 有损环节 | **只有量化（和重排/滤波）是有损的；codec 本身无损** |

取舍原因（官方明确写了对比对象）：

| 对比 | 取舍 |
|------|------|
| vs Draco 一类"最大化压缩比"的库 | 那些库会**打乱顶点/索引顺序** → GPU 上渲染变慢；且不支持自定义量化格式，加载后要重量化，引入额外误差 |
| vs zstd/Oodle 通用压缩 | 通用压缩器**不利用顶点/索引的冗余**，压缩比不理想 |
| meshopt 的选择 | 做一层"保留顺序、可被通用压缩继续压"的轻量编码：想快就直接存编码结果；想小就再叠 zstd/Oodle |

⚠️ 三条硬约束：
1. 顶点编码器假设输入**已 vertex-fetch 优化 + 已量化**；
2. **padding 必须清零**，否则编码器要存未初始化数据；
3. **stride 必须是 4 的倍数**。

### 3.7 Meshlet 生成与簇包围体

```c
const size_t max_vertices  = 64;
const size_t max_triangles = 126;   // v0.25 之前必须是 4 的倍数（常见 124）
const float  cone_weight   = 0.0f;  // 不用锥形剔除就设 0

size_t max_meshlets = meshopt_buildMeshletsBound(indices.size(), max_vertices, max_triangles);
std::vector<meshopt_Meshlet> meshlets(max_meshlets);
std::vector<unsigned int>    meshlet_vertices(indices.size());
std::vector<unsigned char>   meshlet_triangles(indices.size());

size_t meshlet_count = meshopt_buildMeshlets(
        meshlets.data(), meshlet_vertices.data(), meshlet_triangles.data(),
        &indices[0], indices.size(),
        &vertices[0].px, vertices.size(), sizeof(Vertex),
        max_vertices, max_triangles, cone_weight);

// 必做：裁剪超额分配
const meshopt_Meshlet& last = meshlets[meshlet_count - 1];
meshlet_vertices.resize(last.vertex_offset + last.vertex_count);
meshlet_triangles.resize(last.triangle_offset + ((last.triangle_count*3 + 3) & ~3));
meshlets.resize(meshlet_count);

// 推荐：簇内局部优化
for (auto& m : meshlets)
    meshopt_optimizeMeshlet(&meshlet_vertices[m.vertex_offset],
                            &meshlet_triangles[m.triangle_offset],
                            m.triangle_count, m.vertex_count);

// 簇包围体（剔除用）
meshopt_Bounds b = meshopt_computeMeshletBounds(
        &meshlet_vertices[m.vertex_offset], &meshlet_triangles[m.triangle_offset],
        m.triangle_count, &vertices[0].px, vertices.size(), sizeof(Vertex));
```

| 函数 | 用途 | 取舍 |
|------|------|------|
| `meshopt_buildMeshlets` | 默认；在 V/T 限制下平衡**拓扑效率**（簇内顶点复用）与**剔除效率**（半径、方向发散） | 离线处理首选，**即使不用锥剔除也推荐** |
| `meshopt_buildMeshletsScan` | 从顶点缓存优化后的 IB 出发，贪心聚合连续三角形直到超上限 | **加载期**处理更快；质量略低 |
| `meshopt_buildMeshletsFlex` | 给 `min_triangles`/`max_triangles` 两个上限 + `split_factor`（推荐 2.0） | 层级 LOD / 高级剔除要空间局部性时用；⚠️ Bound 要用 `min_triangles` 算 |
| `meshopt_buildMeshletsSpatial` | 用 **SAH** 递归划分，产出 RT 友好的簇（`min_tri=16, max_tri=64, fill_weight=0.5`） | 光线追踪簇加速结构；与光栅化诉求相反 |
| `meshopt_optimizeMeshlet` | 簇内再排一次顶点/三角形顺序 | 提高簇内局部性 → 影响 Mesh Shader 的 gather 是否合并 |
| `meshopt_computeMeshletBounds` | 计算包围球 + 法线锥 | 剔除数据的唯一来源 |

取舍原因（官方原话的要点）：
- 默认 builder 会**为了填满簇而合并空间上不相连的区域** —— 这对顶点复用有利、对锥形剔除不利；所以才需要 `Flex`（牺牲填充率换空间局部性）和 `cone_weight`（把形状纳入目标函数）；
- 光栅化与光线追踪的最优簇形状**相反**：光栅化要"顶点数限制内塞满三角形"，RT 要"最小化 ray-triangle 求交的空间划分"。所以给了两套 builder。

### 3.8 效率分析器（把"优化"变成可回归的数字）

| 分析器 | 主指标 | 最优 / 最坏 |
|--------|--------|------------|
| `meshopt_analyzeVertexCache` | ACMR（顶点调用÷三角形数）、ATVR（着色器调用÷顶点数） | ACMR 规则网格 ≈0.5、最坏 3.0；ATVR 最优 1.0 |
| `meshopt_analyzeVertexFetch` | overfetch（读取字节 ÷ VB 总字节） | 1.0 |
| `meshopt_analyzeOverdraw` | overdraw（PS 调用 ÷ 覆盖像素，多正交相机采样） | 1.0 |
| `meshopt_analyzeCoverage` | 各主轴覆盖率 | 用于衡量简化后的轮廓变化、以及按视点估计覆盖 |

用法建议：**进 CI**。每次资产管线改动后跑一遍，ACMR/overfetch 劣化超阈值就告警。

---

## 4. 集成到引擎的注意事项

| 事项 | 说明 |
|------|------|
| 顶点格式 | 库只认 `stride` 和"位置在结构体里的偏移"，属性语义由你决定 |
| 二进制等价 | ⚠️ padding 清零（影响 remap 与 encode 两处） |
| 分配器 | 默认用 `operator new`；可 `meshopt_setAllocator(malloc, free)`；⚠️ 分配失败要抛异常或 abort，用 malloc 不算安全 |
| 栈占用 | 所有算法栈占用上界 **32 KB** |
| 解码器 | `meshopt_decode*` **不分配内存**，完全在给定缓冲区内工作；可直接写进 write-combined 内存 |
| 版本兼容 | 旧版编码的数据总能被新版解码；新版编码要被旧版解码需**显式 pin 版本**（`meshopt_encodeVertexVersion` / `meshopt_encodeIndexVersion`） |
| Web / 移动端 | 提供 `js/meshopt_decoder.js`（WASM）；索引 codec 只支持 triangle list，条带/线列要用 `encodeIndexSequence` |
| glTF | 同一套编码作为 `EXT_meshopt_compression` / `KHR_meshopt_compression` 扩展落地 |

---

## 5. 必须纠正的说法

| 说法 | 事实 |
|------|------|
| "meshoptimizer 是运行时库" | 是**离线/加载期**库；运行时只用到 decoder（且 decoder 也不是必须放在运行时） |
| "encode 是有损压缩" | codec **无损**；有损的是你之前做的 quantize / simplify / 滤波 |
| "顺序无所谓，随便调" | 顺序错了收益归零甚至为负：`optimizeVertexFetch` 必须在 `buildMeshlets` 前；`encode` 必须最后 |
| "ACMR 越低渲染一定越快" | 分析器是**近似模型**；它可能与实际 GPU 行为不一致，只能用于回归对比 |
| "optimizeOverdraw 免费提升" | 它用 ACMR 劣化换 overdraw；有 depth prepass 时收益可能为零 |
| "buildMeshlets 产出可以直接上传" | 数组按最坏情况超额分配，**必须裁剪** |
| "简化误差是世界单位" | 默认是**相对网格尺度归一化**的；要世界单位需乘 `meshopt_simplifyScale` 或用 `ErrorAbsolute` |
| "cone_weight 越大越好" | 它牺牲顶点复用换剔除率；不用锥剔除时必须为 0 |

---

## 6. 小结

| 要点 | 说明 |
|------|------|
| 核心理念 | 离线一次、近似可测、顺序敏感、零依赖、有损显式化 |
| 流水线 | remap → simplify(LOD) → vertexCache → overdraw → vertexFetch → quantize → encode → meshlet + bounds |
| 顺序红线 | vertexFetch 在 meshlet 前；encode 在最后；buildMeshlets 在 vertexCache 后 |
| 三个指标 | ACMR（顶点缓存）、overfetch（内存）、overdraw（像素）；全部"越小越好"，但互相有冲突 |
| 简化取舍 | 保拓扑（`simplify`，复用 VB、保 UV 缝）vs 不保拓扑（`Sloppy`，快 5–6×） |
| 误差语义 | 相对网格尺度归一化；`lod_error` + 屏幕空间误差公式 → 逐簇 LOD 选择 |
| 压缩取舍 | 不打乱顺序 → 可被 GPU 高效渲染；可再叠通用压缩；2–4×（顶点）/ 5–6×（索引） |
| meshlet builder | 默认=填满（复用优先）；Flex=空间局部（层级 LOD）；Spatial=SAH（RT）；Scan=加载期快 |
| 必做后处理 | 裁剪超额数组 + `meshopt_optimizeMeshlet` + `computeMeshletBounds` |
| 工程纪律 | padding 清零、stride 4 的倍数、pin 编码版本、分析器进 CI |

---

*上一篇：[03-Meshlet结构设计与权衡](03-Meshlet结构设计与权衡.md) | 下一篇：[05-描述符与Bindless统一DrawCall](05-描述符与Bindless统一DrawCall.md)*
