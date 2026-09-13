# 移动端 GPU 架构 — 分箱 Binning 机制详解

> 本文档是「移动端 GPU 架构」系列文档的第 **2** 篇。本篇回答三个最常被问到的问题：分箱是不是必然步骤、分的是顶点还是图元、分箱在管线的哪个位置——并给出三厂商各自的准确答案。

---

## 1. 分箱的本质

### 1.1 一句话定义

**分箱（Binning / Tiling）是在几何阶段把「图元」按屏幕区域归类，生成「每一块包含哪些图元」的索引列表的过程。**

```
      屏幕（帧缓冲）
┌─────┬─────┬─────┬─────┐
│ T00 │ T01 │ T02 │ T03 │        每个格子 = 一个 Tile / Bin
├─────┼─────┼─────┼─────┤
│ T10 │ T11 │ T12 │ T13 │
├─────┼─────┼─────┼─────┤
│ T20 │ T21 │ T22 │ T23 │
└─────┴─────┴─────┴─────┘
        ▲
        │  四个三角形散布在屏幕上
        │
   ┌────┴────────────────────────────────────────────────┐
   │  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
   │  │  大三角   │  │  小三角   │  │  另一个   │  ...      │
   │  │ 跨 4 个块 │  │ 只在 1 块 │  │  小三角   │           │
   │  └──────────┘  └──────────┘  └──────────┘           │
   └─────────────────────────────────────────────────────┘
        │
        ▼  分箱的产物：每个 Tile 一份图元索引列表
   ┌──────────────────────────────┐
   │ T00: [P0, P1]                │
   │ T01: [P0, P1, P3]            │
   │ T02: [P0, P3]                │
   │ T03: []                      │   ← 稀疏：没被覆盖的块是空的
   │ T10: [P0, P2]                │
   │ ...                          │
   └──────────────────────────────┘
```

### 1.2 分箱服务于什么

分箱本身**不产生任何像素**，它纯粹是一个**数据重排**过程。它存在的唯一理由是：

> 让「一块屏幕的数据」小到能塞进片上 SRAM，从而让片元阶段的读写全部发生在片上，而不是主存。

```
没有分箱（IMR）                    有分箱（TBR/TBDR）
─────────────────                ─────────────────
片元着色时：                       片元着色时：
  深度读 ← 主存 32 B/px            深度读 ← 片上 SRAM
  深度写 → 主存                     深度写 → 片上 SRAM
  颜色读 ← 主存                     颜色读 ← 片上 SRAM
  颜色写 → 主存（或混合）            颜色写 → 片上 SRAM
                                  块结束时：Resolve → 主存（一次性线性写）
```

**代价对比**（1080p，一个覆盖全屏的物体，假设平均 1.5 倍 overdraw）：

| 项 | IMR 主存流量 | TBR 主存流量 |
|----|------------|-------------|
| 深度读 | ~207 万 × 4 B × 1.5 ≈ 12 MB | 0 |
| 深度写 | ~207 万 × 4 B ≈ 8 MB | 0 |
| 颜色读写 | ~207 万 × 4 B × 2 ≈ 17 MB | 0 |
| Resolve | 0 | ~8 MB（线性写） |
| **合计** | **~37 MB/帧 ≈ 2.2 GB/s @60fps** | **~8 MB/帧 ≈ 0.5 GB/s @60fps** |

这就是分箱存在的意义——**把随机读写变成一次线性写**。

---

## 2. 分箱的粒度：分的是图元，不是顶点

### 2.1 为什么必须按图元分

▲ 这是资料中最容易含糊的一点。先明确结论：

**分类的对象是图元（Primitive，通常是三角形）。但分箱阶段必须同时运行「位置着色」，因此它极度关心顶点。**

```
┌───────────────────────────────────────────────────────────────────────┐
│  为什么按「图元」分，而不是按「顶点」分？                                 │
│                                                                       │
│  前提：光栅化的输入是**完整的图元**，不是孤立的顶点。                      │
│                                                                       │
│  反证：假设按顶点分                                                    │
│    三角形 ABC 跨越 Tile0 和 Tile1                                     │
│      A 的屏幕坐标落在 Tile0                                           │
│      B、C 的屏幕坐标落在 Tile1                                        │
│                                                                       │
│    若把 A 分给 Tile0、B/C 分给 Tile1：                                 │
│      → Tile0 只有 1 个顶点，构不成三角形，无法光栅化                      │
│      → Tile1 只有 2 个顶点，同样构不成                                   │
│      → 三角形在两边都丢失                                              │
│                                                                       │
│    若把 A、B、C 都复制给两个 Tile：                                     │
│      → 顶点数据被复制，位置属性写入带宽翻倍                               │
│      → 而且仍然需要「哪些顶点组成哪个三角形」的拓扑信息                     │
│                                                                       │
│  结论：分类的最小单位必须是「不可再分的完整几何体」= 图元                   │
└───────────────────────────────────────────────────────────────────────┘
```

### 2.2 但分箱阶段必须跑位置着色

⚠️ **这一点资料完全没提，而它恰恰是移动端最重要的优化依据。**

分箱需要知道「图元在屏幕上的位置」。屏幕位置从哪来？**必须算出来。**

以 ARM Mali 的 IDVS（Index-Driven Vertex Shading）为例，ARM 官方给出的流水线是：

```
Indices ─▶ Index Reader ─▶ Position Cache Fetch ─▶ 【Position Shading】
                                                        │
                                                        ▼
                              Primitive Assembly ─▶ Culling ─▶ Tiling
                                                                 │
                                                                 ▼
                                                            Tile List
                                                                 │
                                                    （对可见图元）
                                                                 ▼
                                                         【Varying Shading】
```

ARM 官方文档（Mali Offline Compiler User Guide §2.7.4）原文：

> "In the IDVS pipeline, vertex shaders are compiled into **two binaries**: A **position shader**, which computes only the position output. A **varying shader**, which computes the remaining non-position outputs. **The position shader runs for every vertex, but the varying shader only runs for vertices that are part of a visible primitive that survives culling.**"

**这就意味着顶点着色器在移动端上的实际执行模型是**：

| 阶段 | 执行范围 | 输出 |
|------|---------|------|
| Position Shading | **所有顶点**（含最终被剔除的） | `gl_Position`（clip space）+ 图元装配所需的拓扑 |
| Varying Shading | **只有存活下来的可见图元的顶点** | UV、法线、世界坐标、切空间等全部 varying |

### 2.3 三个直接可用的优化

```
┌────────────────────────────────────────────────────────────────────────┐
│  优化 ① 顶点缓冲分离（Split / Non-interleaved Stream）                   │
├────────────────────────────────────────────────────────────────────────┤
│  ❌ 交错：struct V { float3 pos; float2 uv; float3 nrm; float3 tan; };  │
│           = 44 字节/顶点                                                │
│           分箱阶段只需要 pos，却拉入 44 字节 → 32 字节是纯浪费            │
│                                                                        │
│  ✅ 分离：                                                              │
│     binding 0: float3 pos        → 12 字节，分箱阶段只读这个             │
│     binding 1: float2 uv         ┐                                     │
│     binding 2: float3 nrm        ├ 只有可见顶点才读                     │
│     binding 3: float3 tan        ┘                                     │
│                                                                        │
│  官方原话（ARM）："To get the most benefit from the Bifrost geometry   │
│  flow it is useful to deinterleave packed vertex buffers partially"    │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│  优化 ② 绝不在 gl_Position 的计算路径上做纹理采样                        │
├────────────────────────────────────────────────────────────────────────┘
│  ❌ float3 local = vertices[i].pos                                     │
│              + Texture2D(dispMap, uv).xyz * scale;   // ← 灾难         │
│     gl_Position = MVP * float4(local, 1.0);                            │
│                                                                        │
│     后果：Position Shader 里出现纹理取指                                 │
│           → 分箱阶段被迫访问纹理 → 带宽 + 延迟双重恶化                    │
│           → Qualcomm 官方列为 Direct Mode 触发条件之一                   │
│                                                                        │
│  ✅ 若必须要顶点位移：改用 compute shader 预计算（在 binning 之前完成）    │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│  优化 ③ 控制 varying 数量与精度                                          │
├────────────────────────────────────────────────────────────────────────┘
│  Varying Shading 的输出要占"后变换顶点缓冲"的空间与带宽。                  │
│  - 砍掉不必要的 varying 输出（编译期能删就让编译器删）                      │
│  - 合并：两个 vec2 → 一个 vec4                                         │
│  - 降精度：mediump (fp16) 在支持的平台上带宽直接减半                       │
│  - Adreno 的 unified shader 中 mediump 为 16-bit，highp 为 32-bit      │
└────────────────────────────────────────────────────────────────────────┘
```

### 2.4 位置着色与「VS 执行两次」的关系

一个常见的困惑：「我的 VS 会不会跑两遍？」

**准确答案**：

```
┌────────────────────────────────────────────────────────────────────────┐
│  严格说，硬件把 VS 编译成两个变体（Position / Varying），不是"同一个 VS    │
│  跑两次"。所以不存在"整段 VS 代码被执行两次"的语义。                        │
│                                                                        │
│  但对你写的那一份 VS 源码而言：                                           │
│    - gl_Position 的计算路径  → 对所有顶点执行（一次）                      │
│    - 其他输出（varying）     → 只对可见图元的顶点执行（一次）                │
│    - 如果某段代码同时在两条路径上（例如 worldPos 参与了 gl_Position         │
│      的计算，又被输出为 varying），它**可能**会被执行两次                    │
│                                                                        │
│  ARM 工程师在社区中的回应：                                               │
│    "The position needed for other uses (e.g. world position for         │
│     lighting) is rarely the same position used for clip and cull        │
│     (clip-space position, after perspective projection). These would    │
│     normally be different computations, even in a non-split flow.       │
│     If there is truly duplicate computation, then it may get repeated,   │
│     but it's pretty rare in practice."                                  │
└────────────────────────────────────────────────────────────────────────┘
```

**实战含义**：真正需要警惕的是**顶点动画（VERTEX ANIMATION）**这类场景——顶点位置依赖纹理采样（烘焙动画 / 骨骼贴图 / 流体 UV 偏移）。这类计算会落在 Position Shader 上，且会直接拖慢分箱阶段。移动端解法：

| 场景 | 移动端推荐做法 |
|------|--------------|
| 骨骼动画 | 骨骼矩阵放 UBO/SSBO（不采样纹理）；顶点索引直接进 UBO |
| 顶点烘焙动画 | 用 compute shader 在帧前预计算到 SSBO，VS 只读 SSBO |
| 顶点位移（地形/水波） | 用 compute 预计算，或接受分箱阶段变慢但确保不在 Direct Mode 触发条件下 |
| GPU-Driven 剔除 | compute 阶段完成 culling，用 indirect draw 提交，让分箱只处理存活图元 |

### 2.5 后变换顶点缓存（Post-Transform Vertex Cache）

分箱阶段有一个关键硬件：**后变换顶点缓存**。它保存最近位置着色过的顶点位置，避免共享顶点被重复着色。

ARM 官方性能计数器文档描述：

> "This pipeline uses a **post-transform vertex cache**, which contains the positions of recently shaded vertices, to avoid reshading vertices that are common to multiple primitives more than once. Poor temporal locality of index reuse in the index buffer can result in a vertex being shaded multiple times, because it can be evicted from the cache before it can be reused."

**官方给出的可量化目标**：

| 网格质量 | 每三角形的顶点着色次数 |
|---------|-------------------|
| 优秀（高顶点复用） | **< 1.5 次** |
| 一般 | 2–3 次 |
| 差（无复用 / 索引有空洞） | 3 次或以上（含重着色与冗余索引） |

⚠️ ARM 还给出了两条容易忽视的坑：

1. **索引范围不能有空洞**：「Reduce redundant shading by ensuring meshes use every index between the min and max index, without any holes.」
2. **着色请求按每 4 个连续索引一组提交**：「Unused index locations may be shaded if they are adjacent to used index locations.」

**对策**：用 `meshoptimizer` 之类的工具做顶点缓存优化（Vertex Cache Optimization），并把索引重排到无空洞的紧凑区间。

---

## 3. Binning Pass 的输入、输出与成本账本

### 3.1 Adreno 的 Binning Pass 产物

Qualcomm 在 SIGGRAPH 2015 的架构演讲中给出明确表述：

> "The only output of Binning Pass is a **compressed 1 bit/primitive 'visibility stream'**. Avoids write bandwidth for transformed positions and overflow/management of transformed VBO."

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Adreno Binning Pass 的账本                                              │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  输入（读主存）：                                                          │
│    · 索引缓冲                (indexCount × 索引宽度)                       │
│    · Position Stream         (vertexCount × 位置属性宽度)                  │
│    · 常量/Uniform 数据                                                    │
│                                                                          │
│  输出（写主存）：                                                          │
│    · Visibility Stream       ≈ 1 bit / 图元（压缩后）                      │
│        100 万三角形 → 约 125 KB 的写流量                                   │
│                                                                          │
│  对比：如果不用 visibility stream，而是存"变换后的顶点"：                     │
│    100 万图元 × 3 顶点 × 12 字节（clip space float3）≈ 36 MB              │
│    → 相差约 290 倍。这就是 Adreno 选择压缩可见性流的理由。                    │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.2 PowerVR 的 Parameter Buffer

PowerVR 的做法不同——它在系统内存中维护一个 **Parameter Buffer（PB）**：

| 内容 | 说明 |
|------|------|
| **Vertex Data** | Tiling Accelerator 传递过来的**所有顶点相关数据** |
| **Primitive Lists** | 「哪些图元属于哪个 tile」的列表 |

PowerVR 官方关于带宽的两条关键注释：

```
              参数缓冲（系统内存）
                     │
        ┌────────────┴────────────┐
        ▼                         ▼
   ┌─────────┐              ┌──────────────┐
   │ (*1)    │              │ (*2)         │
   │ 只取位置 │              │ 法线/纹理等   │
   │ 数据推进 │              │ 只为可见片元取 │
   └─────────┘              └──────────────┘

官方原文：
  (*1) "Only position data is retrieved to perform the next step, saving bandwidth"
  (*2) "Normal data, texture data, et al are only retrieved for what's visible"
```

**这就是 TBDR 相对 TBR 的带宽优势来源**：PowerVR 的 HSR 是 pixel-perfect 且顺序无关的，所以它可以**在真正着色之前**就确定哪些片元可见，从而只为可见片元拉取法线/纹理数据。

⚠️ **PB 满时会怎样？** PowerVR 的机制是 flush 一次 render。官方说明：

> "What happens when the PB is full? A render is flushed. Flushed renders benefit from HSR performed up to that point (HSR 效果不可能是全局最优的). Previously flushed data must be retrieved from the frame buffer for successive tile renders."

**含义**：如果场景图元数量极大，PB 溢出会导致 HSR 的收益被打断。不过官方也说明「Highly unlikely — Big enough you should never hit it」，且**某些平台上 PB 大小可由开发者调整**。

### 3.3 三厂商 Binning 阶段对比

| 维度 | Qualcomm Adreno | ARM Mali | Imagination PowerVR |
|------|----------------|----------|---------------------|
| 分箱硬件 | 专用分箱前端（可与渲染并行） | Tiler（与 shader core 流水重叠） | Tiling Accelerator (TA) |
| 位置着色 | ✅ Binning Pass 运行 position-only VS | ✅ Position Shader（IDVS） | ✅ TA 内做 clip/project/cull |
| 剔除时机 | 分箱时做背面剔除 + 记录可见性 | Culling 在 Position Shading 与 Tiling 之间 | TA 做 clip/project/cull |
| 分箱产物 | 压缩 Visibility Stream（≈1 bit/图元） | Tile List（按 tile 分组的图元索引） | Parameter Buffer（顶点数据 + Primitive Lists） |
| 产物存放 | 系统内存 | 系统内存 | 系统内存 |
| 完整 VS 时机 | Render Pass 内，只对可见图元 | Varying Shading，只对可见图元 | TSP 阶段，**只为可见片元**拉取数据 |
| 微三角形剔除 | 有 | ✅ Bifrost 起在 tiler 阶段可剔除 | 有 |
| Tile 尺寸 | 可变（驱动决定） | 名义 16×16，可用更大的 2 的幂 | Series 6/7 = 32×32 |

---

## 4. 跨 Tile 的三角形怎么处理

### 4.1 官方答案：不裁剪，重复光栅化

⚠️ **这是资料中一个重要的遗漏点，且反直觉。**

Qualcomm 官方文档原文：

> "If a triangle spans multiple tiles in binning mode, **the full triangle will be rasterized per tile – there are no added vertices at tile boundaries**. Therefore, **many triangles much larger than the bin size in screenspace can be inefficient.**"

```
┌──────────────────────────────────────────────────────────────────────┐
│  一个大三角形跨 4 个 Tile 时实际发生的事                                 │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│        ┌─────────┬─────────┐                                          │
│        │  T00    │  T01    │    该三角形被登记到 4 个 tile 的列表中     │
│        │    ╲    │   ╱     │                                          │
│        ├─────────┼─────────┤    T00/T01/T10/T11 各自都有自己的          │
│        │        ╲│╱        │    Visibility Stream 条目               │
│        │  T10    │  T11    │                                          │
│        └─────────┴─────────┘                                          │
│                                                                      │
│  渲染 T00 时：光栅化器拿到完整的三角形，只保留落在 T00 内的片元           │
│  渲染 T01 时：光栅化器**再拿到同一个完整三角形**，只保留落在 T01 内的片元  │
│  渲染 T10、T11：同理                                                    │
│                                                                      │
│  ★ 关键：三角形没有被"切开"，顶点也没有增加。                             │
│    代价是：这个三角形的**光栅化设定(setup)工作被做了 4 次**。             │
└──────────────────────────────────────────────────────────────────────┘
```

### 4.2 为什么这很重要

| 场景 | 后果 |
|------|------|
| 一个覆盖全屏的超大三角形（例如天空盒、全屏地形、大型水面） | 在**每一个**被覆盖的 tile 里都要重复做一次图元设定。1080p + 16×16 tile = 8100 个 tile → 这个三角形的 setup 做了 8100 次 |
| 海量微三角形（每个不到 1 像素） | 每个都要走一遍分箱的包围盒计算 + 写 1 bit 可见性流，本身几乎不产生像素 |
| 中等尺寸、合理复用的网格 | 最优区间 |

**这就是「移动端两头都怕」的原因**：

```
        分箱成本
            ▲
            │  ╲                            ╱
            │   ╲                          ╱
            │    ╲                        ╱
            │     ╲______________________╱
            │
            └──────────────────────────────────────▶ 图元在屏幕上的尺寸
              极小(亚像素)      合理区间        极大(跨大量 tile)
              ↑                                      ↑
        元数据开销主导                        重复光栅化 setup 主导
        （分箱条目 + 包围盒计算）                （每个 tile 都做一次）
```

### 4.3 可操作的对策

| 问题 | 对策 |
|------|------|
| 超大三角形 | 在 CPU/编辑器侧做**镶嵌（subdivision）**，把大三角形切成比 tile 略大的网格。地形、水面、天空盒应避免单个巨型三角形 |
| 亚像素微三角形 | LOD 降级、微三角形剔除（`VK_EXT_mesh_shader` / `VK_EXT_primitive_shading_rate` 或 Mali tiler 自带的微三角形剔除）、简化网格 |
| 城市级建筑群 | 层次化剔除（HLOD），确保提交到分箱的图元总数受控 |
| 大量实例化小物件 | GPU-Driven + indirect draw，在 compute 阶段完成剔除与 LOD 选择 |

---

## 5. 并发分箱：为什么「分箱时间」常常看不到

### 5.1 机制

Adreno 的图形前端包含**两条并行的管线**：

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Adreno 图形前端                                                          │
│                                                                          │
│   ┌──────────────────────┐          ┌──────────────────────────────┐     │
│   │  Binning 前端         │  ←并行→  │  渲染后端                     │     │
│   │  · Position Shading  │          │  · Raster                  │     │
│   │  · 图元装配/剔除       │          │  · Shader Cores            │     │
│   │  · 生成 Visibility    │          │  · ROP                     │     │
│   │    Stream            │          │  · Resolve                 │     │
│   └──────────────────────┘          └──────────────────────────────┘     │
│              ▲                                    ▲                      │
│              │                                    │                      │
│        Surface N+1 的分箱 与 Surface N 的渲染 重叠执行                      │
└──────────────────────────────────────────────────────────────────────────┘
```

### 5.2 什么会破坏并发分箱（Qualcomm 官方列出的三条）

| 破坏条件 | 说明 | 对策 |
|---------|------|------|
| **顺序 render pass 之间存在依赖** | 如果 Pass C 的分箱需要 Pass B 的输出作为顶点着色器输入，就迫使分箱同步执行 | 重新审视 render pass 设置与 barrier，尽量切断 Pass 之间的顶点输入依赖 |
| **帧内对同一 Z-buffer 反复 clear** | clear 定义了依赖，**阻止所有使用该 Z-buffer 的 render pass 或 compute 操作的并发分箱** | 跨多个 render pass 复用同一个 Z-buffer，不做 clear、不做 invalidate |
| **VSYNC 受限且首个 surface 承担全部几何负载** | 第一个 surface 的分箱成为串行瓶颈 | 在该 pass 之前安排一些独立工作，让独立工作与首个 surface 的分箱并行 |
| （备选方案）| 如果实在做不到复用 Z-buffer | 给每个 render pass 分配独立的 Z-buffer 也能允许并发分箱，但会增加内存与带宽开销 |

⚠️ **注意这里存在一组紧张关系**：

```
        规则 A：颜色附件必须 clear 或 invalidate
                （否则驱动从主存读回 tile buffer）
                          ⚡
        规则 B：Z-buffer 跨 pass 复用且不 clear
                （否则破坏并发分箱）
```

**两条规则不矛盾，只是作用对象不同**：

| 附件类型 | 规则 |
|---------|------|
| 颜色附件（需要写回主存的） | 每次使用都要明确 `LOAD_OP_CLEAR` 或 `LOAD_OP_DONT_CARE` |
| 深度/模板附件（跨 pass 复用的） | 避免帧内重复 clear；跨 pass 保持 `LOAD_OP_LOAD` |
| 深度/模板附件（本帧之后不再使用） | `STORE_OP_DONT_CARE`（或用 `VK_ATTACHMENT_STORE_OP_NONE_QCOM`） |

### 5.3 对 Profiler 解读的影响

因为并发分箱的存在：

> **「Binning Pass 耗时占比高」不等于「分箱阻塞了渲染」。** 只有当分箱管线本身成为关键路径时，它才真正影响帧时间。看到高占比时，先确认并发分箱是否被上面的三条规则之一破坏了。

---

## 6. 分箱能否跳过

### 6.1 按平台

| 平台 | 能否跳过分箱 | 说明 |
|------|------------|------|
| Adreno | ✅ 能（Direct Mode） | 驱动启发式逐 surface 决定；也有 Binned Direct Mode 的半程方案 |
| Mali | ⚠️ 基本不能 | TBR 是核心架构。但分箱开销与渲染流水重叠 |
| PowerVR | ❌ 不能 | 真 TBDR，HSR 依赖分箱产物 |
| Apple GPU | ❌ 不能 | TBDR |

### 6.2 Adreno 的三种模式对照

| 模式 | 是否有 Binning Pass | 渲染方式 | 适用场景 |
|------|------------------|---------|---------|
| **Direct Mode** | ❌ 无 | 等价 IMR，直接渲染到帧缓冲 | 顶点/draw 数少；VS 中纹理采样多；使用 tessellation / GS；小 surface |
| **Binned Mode** | ✅ 有 | 逐 bin 渲染进 GMEM，最后 Resolve | 图元多、surface 大、带宽受限的常态场景；**移动端默认最优** |
| **Binned Direct Mode** | ✅ 有（仅可见性 Pass） | 先分箱剔除背面 → 切 Direct 渲染 | 想拿到「免费的深度预通道」收益但不需要完整 binning 的场景 |

### 6.3 官方启发式规则

Qualcomm《Adreno GPU on Mobile: Best Practices》原文：

> "The driver heuristics that determine when to run binned or direct mode are **not exposed to the developer**, but generally these scenarios trigger direct mode:
> - High ratio of texture samples in vertex shaders to vertices
> - Small number of vertices and/or draws
> - Use of tessellation or geometry shaders"

⚠️ **重要**：这些规则**不可查询、不可配置**。所以在 Adreno 上调优时，需要**通过实验确认**驱动选择了哪种模式，而不是假设。可通过 Snapdragon Profiler 的 surface 级指标观测。

---

## 7. 分箱成本的影响因素全景表

| 类别 | 因素 | 影响方向 | 可操作对策 |
|------|------|---------|-----------|
| **图元数量** | 提交到分箱的图元总数 | 每个图元都要算包围盒 + 写 ~1 bit 可见性流 | LOD、GPU-Driven 剔除、合并绘制、减少子网格 |
| **图元尺寸** | 屏幕空间尺寸 vs tile 尺寸 | 远大于 tile → 每个 tile 重复 setup；远小于 tile → 元数据开销主导 | 大三角形镶嵌、微三角形剔除 |
| **顶点数量** | Position Shading 覆盖的顶点数 | 全额开销（含最终被剔除的顶点） | 减少顶点、索引紧凑无空洞、顶点缓存优化 |
| **顶点复用率** | 每三角形的顶点着色次数 | 目标 < 1.5 次/三角形 | meshoptimizer 顶点缓存优化 |
| **Position Shader 复杂度** | `gl_Position` 计算路径的指令数与访存 | 直接线性影响分箱时间 | 避免在位置路径上采样纹理；顶点动画改 compute 预计算 |
| **Position Stream 布局** | 交错 vs 分离 | 交错布局每顶点多读 2–3 倍字节 | 分离顶点流 |
| **Draw Call 数量** | 每个 draw 的分箱开销 | draw 数少 → 可能直接触发 Direct Mode；draw 数极多 → 分箱前端饱和 | GPU-Driven + MDI 合并 |
| **Tile 数量** | 屏幕分辨率 / bin 尺寸 / 附件格式 | bin 变小 → bin 数增加 → 分箱元数据增加 | 降分辨率、VRS、减少 MSAA 采样、减少同帧活跃 RT 数 |
| **并发分箱是否生效** | 依赖链、Z-buffer clear 习惯 | 生效 → 分箱时间被隐藏；失效 → 串行累加 | 切断 pass 间顶点依赖；跨 pass 复用 Z-buffer 不 clear |
| **驱动模式选择** | Direct / Binned / Binned Direct | Direct 模式下无分箱开销但失去 GMEM 收益 | 消除触发 Direct Mode 的三个条件（尤其 VS 纹理采样） |

---

## 8. 优化清单

### 8.1 立即可做（低风险、高收益）

```
□ 1. 顶点缓冲分离：position 单独一个 binding，其余 varying 另放
     → 分箱阶段读带宽按比例下降

□ 2. 检查所有 shader 的 gl_Position 计算路径，确认没有纹理采样
     → 若有，改用 compute 预计算到 SSBO

□ 3. 用 meshoptimizer 做顶点缓存优化，并确保索引区间无空洞
     → 目标：每三角形顶点着色次数 < 1.5

□ 4. 检查 varying 数量与精度，能合并的合并，能用 mediump 的降精度
     → 减少 Varying Shading 的输出带宽

□ 5. 避免单个超大三角形（地形/天空盒/水面）
     → 在资产侧确保三角形屏幕尺寸与 tile 尺寸同量级
```

### 8.2 需要管线配合（中风险）

```
□ 6. 用 GPU-Driven 渲染（compute culling + indirect draw）
     → 让分箱只处理真正存活的图元，同时压低 draw call 数

□ 7. 给所有 render pass 分配独立的深度附件（或跨 pass 复用而不 clear）
     → 允许并发分箱

□ 8. 审查 render pass 依赖链，切断「Pass N+1 的顶点着色器依赖 Pass N 输出」
     → 同样的目的

□ 9. 减少同帧活跃的渲染目标数量
     → 减少 bin 数量（Qualcomm 官方 "Bin Minimization" 手段之一）

□ 10. 若使用 MSAA，降到 2x
     → Qualcomm 官方：MSAAx2 在实践中通常近乎免费
```

### 8.3 验证手段

```
□ 11. Mali 平台：跑 Mali Offline Compiler，逐项检查
        - Position shader / Varying shader 的周期数
        - 是否有 late ZS test / modifies coverage / reads color buffer
        - Position threads per input primitive、Varying threads per input primitive

□ 12. Adreno 平台：Snapdragon Profiler 关注
        - Binning Pass 相关指标及其占比
        - surface 数量与每个 surface 的实际渲染模式
        - 是否出现 Direct Render Mode 警告

□ 13. 跨平台：用 VK_EXT_subpass_merge_feedback 确认 subpass 是否真的被合并
        （合并失败说明驱动无法在片上完成数据传递）
```

---

## 9. 小结

| 要点 | 说明 |
|------|------|
| **分箱的粒度是图元** | 因为光栅化需要完整拓扑；按顶点分会导致跨块三角形在两边都构不成完整图元 |
| **但分箱必须跑位置着色** | Position Shader 覆盖**全部**顶点（含被剔除的），Varying Shader 只覆盖可见图元的顶点。这是移动端最重要的优化依据 |
| **分箱在位置着色之后、完整着色之前** | 不是「顶点着色器之后」，而是夹在两段顶点处理之间 |
| **分箱有主存流量** | Adreno 输出约 1 bit/图元的压缩可见性流；PowerVR 写入 Parameter Buffer。图元数量本身就是成本 |
| **跨 block 的三角形不裁剪** | 完整三角形在每个覆盖的 block 中重复光栅化，不增加顶点。超大三角形低效，微三角形也低效，中间区间最优 |
| **并发分箱让分箱时间通常被隐藏** | 三条破坏条件：pass 间依赖、帧内重复 clear 同一 Z-buffer、VSYNC 受限 + 首 surface 承载全部几何 |
| **分箱能否跳过取决于平台** | Adreno 可以（FlexRender 三档）；Mali/PowerVR/Apple 基本不能 |
| **用可观测的手段替代猜测** | Mali Offline Compiler 的检查项、Snapdragon Profiler 的 surface 级指标，都是可以接进 CI 的确定性信号 |

---

*上一篇：[01-资料准确性审查与勘误](01-资料准确性审查与勘误.md) | 下一篇：[03-逐Tile流水线与GMEM](03-逐Tile流水线与GMEM.md)*
