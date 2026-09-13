# 移动端 GPU 架构 — 逐 Tile 流水线与 GMEM

> 本文档是「移动端 GPU 架构」系列文档的第 **3** 篇。本篇拆解「一个 Tile 从几何到像素的完整生命周期」，讲清片上内存（GMEM / Tile Memory）的容量事实、溢出代价、Resolve 成本，以及 Vulkan 侧的显式控制手段。

---

## 1. 逐 Tile 流水线的完整时序

### 1.1 单 Tile 的生命周期

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  一个 Tile 的完整生命周期（以 Adreno Binned Mode 的 Deferred 管线为例）          │
└──────────────────────────────────────────────────────────────────────────────┘

  【前置】Binning Pass 已完成，Visibility Stream 已写入系统内存
                          │
                          │  调度器取出一个 tile，交给空闲的 shader core
                          ▼
  ╔══════════════════════════════════════════════════════════════════════════╗
  ║ 阶段 1：取可见图元                                                          ║
  ║   读 Visibility Stream → 只处理落在本 tile 的图元                            ║
  ║   ⚠️ 这一步有主存读（Visibility Stream）                                    ║
  ╚══════════════════════════════════════════════════════════════════════════╝
                          │
                          ▼
  ╔══════════════════════════════════════════════════════════════════════════╗
  ║ 阶段 2：完整顶点着色（Varying Shading）                                     ║
  ║   算 UV / 法线 / 世界坐标等非位置输出                                        ║
  ║   ⚠️ 只对可见图元的顶点执行                                                 ║
  ║   ⚠️ 有主存读（Varying Stream）                                            ║
  ╚══════════════════════════════════════════════════════════════════════════╝
                          │
                          ▼
  ╔══════════════════════════════════════════════════════════════════════════╗
  ║ 阶段 3：光栅化（Rasterization）                                            ║
  ║   把图元离散化为片元，限定在本 tile 的矩形范围内                               ║
  ║   → 输出 pixel quad（通常 2×2 或 4×4 的片元簇）                             ║
  ╚══════════════════════════════════════════════════════════════════════════╝
                          │
                          ▼
  ╔══════════════════════════════════════════════════════════════════════════╗
  ║ 阶段 4：隐面剔除（HSR / Early-Z / LRZ）                                    ║
  ║   用片上深度缓冲剔除被遮挡的片元                                              ║
  ║   ✅ 全程片上访问，无主存流量                                                ║
  ║   ⚠️ 会被 gl_FragDepth 写入、discard、framebuffer fetch 削弱               ║
  ╚══════════════════════════════════════════════════════════════════════════╝
                          │
                          ▼
  ╔══════════════════════════════════════════════════════════════════════════╗
  ║ 阶段 5：片元着色（Fragment Shading）— 若管线是 Deferred，则为几何 Pass       ║
  ║   写 G-Buffer（albedo / normal / roughness ...）                            ║
  ║   ✅ 全部写进片上 GMEM                                                      ║
  ║   ⚠️ 有主存读（纹理采样）                                                   ║
  ╚══════════════════════════════════════════════════════════════════════════╝
                          │
                          ▼
  ╔══════════════════════════════════════════════════════════════════════════╗
  ║ 阶段 6：光照 Pass（Deferred 的第二阶段）                                     ║
  ║   通过 Input Attachment（local read）读取阶段 5 写入片上的 G-Buffer          ║
  ║   ✅ 像素局部读取，不落主存                                                  ║
  ║   ⚠️ 有主存读（光源数据、阴影图、IBL 纹理）                                   ║
  ╚══════════════════════════════════════════════════════════════════════════╝
                          │
                          ▼
  ╔══════════════════════════════════════════════════════════════════════════╗
  ║ 阶段 7：逐片元操作 + 写回片上颜色缓冲                                          ║
  ║   深度测试 / 模板 / 混合 → 写入片上 color buffer                            ║
  ║   ✅ 片上访问                                                               ║
  ╚══════════════════════════════════════════════════════════════════════════╝
                          │
                          ▼
  ╔══════════════════════════════════════════════════════════════════════════╗
  ║ 阶段 8：Resolve（解析）                                                     ║
  ║   把本 tile 的最终颜色（例如 32×32 像素）打包，通过内存总线写入系统内存          ║
  ║   ⚠️⚠️ 这是本 tile 生命周期内**唯一不可避免的大宗主存写**                      ║
  ║   G-Buffer 不写回（storeOp = DONT_CARE）                                    ║
  ╚══════════════════════════════════════════════════════════════════════════╝
                          │
                          ▼
                    释放 shader core，取下一个 tile
```

### 1.2 并行与重叠：tile 之间不是串行的

资料中「Tile 1 写回时 Tile 2 在做光照、Tile 3 在做几何」的说法方向正确，但需要更准确的模型。

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  实际的时间线（示意图）                                                         │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Binning 前端   ████████████████████                                         │
│                 （Surface N+1 的分箱，与下面并发）                              │
│                                                                              │
│  Tile 处理       ░░▓▓▒▒░░▓▓▒▒░░▓▓▒▒░░▓▓▒▒  ← 多 tile 在硬件流水上重叠         │
│                  tile0 tile1 tile2 tile3 ...                                 │
│                    │     │     │                                              │
│  阶段分布         │     │     └─ 阶段1-8 的不同阶段重叠                        │
│                   │     └─ 上一个 tile 的 Resolve 与本 tile 的取指重叠          │
│                   └─ ...                                                     │
│                                                                              │
│  ⚠️ 关键：不是"每个 shader core 固定绑一个 tile"，而是：                        │
│     · 多 tile 在**时间上重叠**（流水线各级并行）                                 │
│     · GMEM 是**共享资源**，被同时活跃的 tile 共同占用                            │
│     · 同时活跃的 tile 数量由硬件调度与 GMEM 可用量共同决定，不可查询              │
└──────────────────────────────────────────────────────────────────────────────┘
```

**这个模型有几个重要推论**：

| 推论 | 实践含义 |
|------|---------|
| 同时活跃的 tile 数与 GMEM 用量正相关 | GMEM 需求不是「一个 tile 的附件大小」，而是「同时活跃 tile 数 × 单 tile 附件大小」 |
| 每个 tile 的 Resolve 都有固定开销 | tile 越小 → tile 越多 → Resolve 次数越多 → 固定开销线性增长 |
| 阶段 4–7 全在片上 | 所以「片元阶段增加一个附件」的成本主要体现在**片上容量**，而非主存带宽 |

---

## 2. GMEM / Tile Memory 容量事实

### 2.1 已知数字与公开程度

⚠️ **这是业内信息最混乱的一块。** 必须区分「官方公布」与「社区推测」。

| 厂商 / 型号 | 片上内存 | 来源 | 可信度 |
|------------|---------|------|-------|
| Qualcomm Adreno 615 | **512 KB** | Chips and Cheese 独立分析 | 高（第三方实测） |
| Qualcomm Adreno 630 | **1 MB** | 多方社区分析一致 | 中高 |
| Qualcomm Adreno X1（PC） | **3 MB**，带宽 2 TB/s+ | **Qualcomm 官方**（架构深度拆解） | 极高 |
| ARM Mali-G52 | 约 **16 KB**（两 Shader Core 合计） | Chips and Cheese 推算（"tile memory is attached to each Shader Core"，Bifrost 每像素 256 bit tile 存储） | 中 |
| ARM Mali 高端（如 G78） | 每核约 16 KB 量级 | 社区推算，官方未系统公布 | 中低 |
| Imagination PowerVR | 未公开 | — | — |
| Apple GPU | 未公开 | — | — |

### 2.2 关键结论

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ① Adreno 的 GMEM 与 Mali 的 Tile Memory 差一到两个数量级                       │
│                                                                              │
│     Adreno 630:  ████████████████████████████████████████  1 MB              │
│     Adreno 615:  ████████████████████████                512 KB              │
│     Mali G52:    █                                        16 KB              │
│                                                                              │
│     差 64 倍！（1 MB vs 16 KB）                                               │
│                                                                              │
│  ② 因此两者优化重点完全不同                                                     │
│     · Adreno：GMEM 大 → 可以装下胖 G-Buffer → 重点在"别切到 Direct Mode"          │
│     · Mali：  Tile Memory 小 → 装不下胖 G-Buffer → 重点在"极限压缩附件格式"        │
│                                                                              │
│  ③ 所以"移动端 G-Buffer 预算"没有统一答案                                        │
│     按最弱的平台（Mali 小核）设计，才不会在低端机上崩                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 2.3 GMEM 不是 cache

⚠️ 这是一个常被误解的概念。Qualcomm 官方明确说明：

> "Architecturally, GMEM is not just a cache, because it is **separated from the system memory hierarchy**, and the GPU can do almost any operation on memory (**including using it as a cache if necessary**)."

| 维度 | GMEM / Tile Memory | L2 Cache |
|------|-------------------|----------|
| 与主存一致性 | ❌ 不自动保持一致 | ✅ 自动维护 |
| 寿命 | 一个 tile 的渲染周期 | 由替换策略决定，不可预测 |
| 分配方式 | 驱动在 render pass 级别静态规划 | 硬件动态 |
| 可显式分配 | ✅（`VK_QCOM_tile_memory_heap`） | ❌ |
| 用途 | color / depth / G-Buffer 中间结果 / 通用本地内存 | 通用缓存 |
| 可被 compute 使用 | ✅（Adreno X1 起明确支持通用计算负载） | ✅ |

**含义**：GMEM 需要**显式管理其生命周期**。这就是为什么 `loadOp` / `storeOp` 如此重要——它们是你告诉硬件「这个附件在片上从哪来、到哪去」的唯一手段。

---

## 3. Tile / Bin 尺寸：能不能设置

### 3.1 核心结论

| 问题 | 答案 |
|------|------|
| 能在 Vulkan 里设置 tile 尺寸吗？ | ❌ **不能**。核心 Vulkan 无此接口，厂商扩展也没有设置接口 |
| 能查询吗？ | ✅ 能。`VK_QCOM_tile_properties`（Adreno） |
| 尺寸是固定的吗？ | 分厂商：Mali 名义固定（16×16，可上调至更大的 2 的幂）；**Adreno 可变**（驱动按 GMEM + 格式决定）；PowerVR 按系列固定 |

### 3.2 为什么不能设置

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  硬件为什么把 tile 尺寸写死（或交给驱动决定）                                     │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ① 光栅化器的工作粒度与 tile 尺寸绑定                                           │
│     光栅化器一次处理 2×2 或 4×4 的 pixel quad，tile 必须是它的整数倍              │
│                                                                              │
│  ② 内存控制器按固定突发长度工作                                                 │
│     片上 SRAM 的 bank/port 组织、主存的 burst 长度，都按 tile 尺寸设计            │
│                                                                              │
│  ③ tile list 的数据结构按 tile 网格布局                                         │
│     驱动为每帧建立 tile → primitive list 的映射表，尺寸变化会牵动整套元数据        │
│                                                                              │
│  ④ 固定尺寸让驱动可以做静态资源规划                                              │
│     GPU 在命令缓冲构建/管线编译阶段就能算出 GMEM 需求，提前决定 bin 尺寸            │
│                                                                              │
│  → 如果开发者能随意设置 tile 尺寸，硬件与驱动都要做大量动态适配，得不偿失             │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 为什么 1×1 是灾难

资料的结论正确，但推理需要修正。**tile list 是稀疏的**（只有被覆盖的 tile 才有非空列表），所以「1×1 会产生 207 万个列表」这个说法不准确。真实的问题在三处：

| 问题 | 说明 |
|------|------|
| **元数据开销爆炸** | 每个非空 tile 至少需要一个列表头（指针 + 计数）。1080p 下即使只有 10% 的 tile 被覆盖，也是 20 万个列表头。按每头 8 字节算就是 1.6 MB 的元数据——**而且与图元数量无关，纯属噪声** |
| **调度粒度退化** | tile 是调度的最小单位。1×1 意味着调度器要管理 207 万个工作项，调度开销本身超过实际计算 |
| **图元登记开销放大** | 一个小三角形在 1×1 下可能被登记到几十个 tile 中（取决于它覆盖多少像素），而每个 tile 只能产出 1 个像素的有效结果 |
| **Resolve 次数爆炸** | 每个 tile 结束都要 Resolve 一次（即使只有 1 个像素）。Resolve 是打包 + 总线事务，有固定成本 |

⚠️ **注意什么不是问题**：片上 SRAM 的访问不存在「cache line 浪费」的问题（SRAM 是可独立寻址的宽并行阵列）。资料中说「1 个像素要拉入 128 字节 cache line」是对片上访问的误解——那条论述适用于**主存访问**，而 tile 越小反而片访问效率越低的是**调度与元数据**层面。

### 3.4 真正的「缩放 tile」手段

既然不能直接改 tile 尺寸，想减少 tile 数量的官方途径是 **Bin Minimization**（Qualcomm 官方术语）：

| 手段 | 效果 | 副作用 |
|------|------|--------|
| 降低帧缓冲分辨率 | tile 数按面积平方下降 | 画质下降（可用 upscale 缓解） |
| 使用 VRS（含注视点渲染） | 减少片元数量，等效减轻每 tile 负担 | 需要硬件支持；Mali 上需注意 shading rate ≤ 2×2 |
| **减少 MSAA 采样数** | 每像素格式变轻 → bin 可以更大 | MSAAx2 通常近乎免费；大于 2x 收益递减 |
| 减少同时渲染的渲染目标数量 | 每像素格式变轻 → bin 可以更大 | 需要改管线设计 |
| 压缩附件格式（打包 / 降精度） | 同上 | 需要精心设计编解码 |

---

## 4. G-Buffer 尺寸的算术与决策公式

### 4.1 单像素成本表

| 附件 | 格式 | 字节/像素 | 备注 |
|------|------|----------|------|
| Albedo (RGB) + Roughness (A) | `RGBA8_UNORM` | 4 | 铝 + 粗糙度打包，常用 |
| Albedo | `RGBA8_SRGB` | 4 | sRGB 采样需求时 |
| Normal（Octahedron 编码） | `RG16F` | 4 | ✅ 移动端推荐 |
| Normal（原始） | `RGBA16F` | 8 | ❌ 带宽杀手 |
| Normal + Metallic | `RGB10A2` | 4 | 球面编码 + 金属度 |
| Metallic / Roughness / AO | `RGBA8` | 4 | 可进一步打包 |
| World Position | — | **0** | ✅ 用深度重建，不写位置 |
| Depth / Stencil | `D24S8` | 4 | 若不需要模板可用 `D32F` |
| Depth 高精度 | `D32F` | 4 | — |
| Emissive | `RGB10A2` | 4 | — |
| Motion Vector | `RG16F` | 4 | TAA 需要 |

### 4.2 一套移动端实战 G-Buffer 方案

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  方案 A：极致精简（总 16 字节/像素）                                            │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  RT0  RGBA8_UNORM                                                            │
│       R: albedo.r          ← sRGB 分量，若需要可单独处理                        │
│       G: albedo.g                                                           │
│       B: albedo.b                                                           │
│       A: roughness                                                          │
│                                                                              │
│  RT1  RGBA8_UNORM   ← Octahedron normal 用 RG16 会更好，这里退一步用 RGBA8     │
│       RG: octahedron-encoded normal (2 × 8 bit)                             │
│       B:  metallic                                                          │
│       A:  AO / 材质 ID / 自定义                                                │
│                                                                              │
│  DEPTH  D24S8 或 D32F                                                       │
│       → World Position 在光照 Pass 中由深度反投影重建                          │
│                                                                              │
│  合计：4 + 4 + 4 = 12 字节/像素（不含 MSAA）                                   │
└──────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  方案 B：更高画质（总 20 字节/像素）                                            │
├──────────────────────────────────────────────────────────────────────────────┤
│  RT0  RGB10A2   albedo(rgb) + roughness(a)                                  │
│  RT1  RG16F     octahedron normal                                           │
│  RT2  RGBA8     metallic + AO + 材质 ID + 自定义                               │
│  DEPTH D32F                                                                 │
│  合计：4 + 4 + 4 + 4 + 4 = 20 字节/像素                                       │
└──────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  ❌ 反例：桌面惯性方案（总 32+ 字节/像素）                                       │
├──────────────────────────────────────────────────────────────────────────────┤
│  RT0 RGBA16F  albedo      (8 B)  ← 过度精度                                    │
│  RT1 RGBA16F  normal      (8 B)  ← 应有 Octahedron 编码                       │
│  RT2 RGBA16F  worldPos    (8 B)  ← 完全可以从深度重建                          │
│  RT3 RGBA16F  specular    (8 B)                                              │
│  DEPTH D24S8              (4 B)                                              │
│  合计：36 字节/像素                                                            │
│                                                                              │
│  相比方案 A，主存/片上需求是 3 倍。1080p 下每帧多出的 Resolve/读回流量：           │
│    24 字节 × 207 万 = 约 50 MB/帧 ≈ 3 GB/s @60fps（假设要写回）                 │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 决策公式

在确定 G-Buffer 方案前，必须先算这道题：

```
    单 tile 的片上需求
    = Σ(各附件字节/像素) × 采样倍数 × tile 宽 × tile 高

    片上总量需求
    = 单 tile 片上需求 × 同时活跃 tile 数
    + 其他运行时占用（指令、常量、本地数组等）

    判定：
      片上总量需求 ≤ 可用片上内存  →  ✅ 全片上闭环
      片上总量需求 > 可用片上内存  →  ⚠️ 驱动将做出妥协（见下节）
```

⚠️ **「同时活跃 tile 数」不可查询**，是硬件调度与驱动的内部决策。所以这道题只能定性推算，实践中的做法是：**先按最弱目标平台（小 Tile Memory 的 Mali）设计，再往上放宽**。

### 4.4 采样倍数（MSAA）

| MSAA | 片上需求倍数 | 移动端评价 |
|------|------------|-----------|
| 1x | ×1 | Deferred 的唯一选择 |
| 2x | ×2 | Qualcomm 官方：**实践中通常近乎免费**（虽然会产生更多 bin，但额外分箱开销与 Resolve 时间通常能被其他瓶颈隐藏） |
| 4x / 8x | ×4 / ×8 | 与 Deferred 语义冲突；对 Forward 也需谨慎 |

#### 4.4.1 为什么 MSAA 与延迟渲染「语义冲突」

> 上面表格里「与 Deferred 语义冲突」不是指 MSAA 在延迟渲染里跑不起来，而是 **MSAA 的底层假设与延迟渲染的 G-Buffer 数据结构在原理上不兼容**。

**① MSAA 到底在干什么**

MSAA 的本质是「覆盖率按 sample 算，颜色只算一次」：

- 光栅化时一个像素里的 N 个 sample 点，图元覆盖哪些 sample 是**逐 sample** 记录的（边缘抗锯齿的来源）；
- 但**片元着色器只在像素中心跑一次**，输出颜色被**广播**到所有被覆盖的 sample；
- 最后 **Resolve** 把 N 个 sample 的颜色**平均**成 1 个像素颜色。

关键点：**MSAA 的 Resolve 是对「颜色」取平均**，全部前提都是「你存的是可平均的颜色」。

**② 为什么和延迟渲染冲突**

延迟渲染把工作拆成：几何 Pass 写 G-Buffer（MRT：法线 / albedo / 材质 ID / 深度，都是**表面属性而非颜色**），光照 Pass 每像素采样一次 G-Buffer 算最终颜色。冲突有两处硬伤：

| 硬伤 | 说明 |
|------|------|
| G-Buffer 不是颜色，没法平均 | 边缘处 sample 0 在表面 A、sample 1 在表面 B；把两个**不同表面的法线**平均得到指向虚空的法线，把**材质 ID** 平均得到不存在的 1.5 号材质 → 光照 Pass 算出垃圾结果 |
| 逐 sample 覆盖率进不了 G-Buffer | MSAA 靠「逐 sample 记录哪个图元覆盖」从光栅化带到 Resolve；但 G-Buffer 每像素只存一份属性、光照 Pass 每像素只采样一次，sample 级覆盖率被压扁。要真正生效必须 G-Buffer 每 sample 各存一份 + 光照逐 sample 着色（成本 ×N），这等于否定延迟渲染「每像素着色一次」的立身之本 |

所以 **1x 才是 Deferred 的唯一选择**：要么不用 MSAA，要么改用后处理 AA（FXAA / SMAA）只在最终颜色上做。

**③ 移动端的放大效应**

在 TB(D)R 上冲突又叠了一层带宽/片上内存灾难：

- G-Buffer 是 **MRT（多张附件）**，MSAA ×4/×8 让**每张**附件片上需求都 ×N，总体积是 MRT 数 × 倍数；
- 这些本应留在 GMEM / Tile Memory 的中间结果暴涨，直接把 bin 撑大甚至溢出回主存（见第 5 节）；
- Mali 每核 Tile Memory 仅 ~16KB 量级，Adreno 的 GMEM 也会被吃光，驱动被迫 spill 或切 Direct Mode。

⚠️ **关于 2x「近乎免费」的边界**：那是 Qualcomm 针对**最终颜色 Resolve** 的说法（片上 resolve 便宜、额外分箱/Resolve 时间能被掩盖），**绝不等于 G-Buffer 可以用 2x MSAA**。2x 该用在 Forward 的最终颜色或延迟渲染的光照结果上；G-Buffer 本身依旧必须 1x（见清单 C1 与 Vulkan 骨架的 `VK_SAMPLE_COUNT_1_BIT` 注释）。

---

## 5. GMEM 溢出的真实后果

### 5.1 驱动面临的选择

当片上需求超出可用容量时，驱动**不是简单地「溢出」**，而是按优先级做妥协：

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  驱动在命令缓冲构建 / 管线编译阶段就能算出片上需求                              │
│                              │                                               │
│                              ▼                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │ 选项 1：缩小 bin 尺寸（首选）                                             │  │
│  │   bin 变小 → 单 bin 片上需求下降 → 仍在容量内                             │  │
│  │   代价：bin 数量增加 → 分箱元数据增加、Resolve 更碎、并发度变化              │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                              │                                               │
│                              ▼  若仍不够或格式要求不允许                        │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │ 选项 2：把部分附件落到系统内存（Spill）                                    │  │
│  │   中间结果写回主存，用时再读回                                             │  │
│  │   代价：本应省下的带宽全部还回去，且延迟飙升                                 │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                              │                                               │
│                              ▼  若该 surface 被启发式判定不适合 binning          │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │ 选项 3：整个 surface 走 Direct Mode（Adreno）                             │  │
│  │   放弃 binning 与 GMEM，退化为 IMR 行为                                   │  │
│  │   代价：所有带宽优化失效                                                   │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Adreno Direct Mode 的官方触发条件

⚠️ **必须按官方口径理解**，而不是笼统归因于「容量不足」：

| 触发条件 | 说明 | 与容量相关吗 |
|---------|------|------------|
| VS 中纹理采样与顶点数量之比高 | 分箱阶段的纹理访问让 binning 不划算 | ❌ 无关 |
| 顶点数和 / 或 draw 数少 | binning 的固定开销摊不薄 | ✅ 部分相关（小 surface 更易命中） |
| 使用 tessellation 或 geometry shader | 几何放大发生在分箱阶段之后，binning 难做静态规划 | ❌ 无关 |
| （未明示但合理）大 surface + 重格式 | 驱动倾向放弃 binning | ✅ 相关 |

**这说明一件事**：把 Direct Mode 警告全部归因于「G-Buffer 太大」是错的。**要先排查前三条**，尤其是「VS 里的纹理采样」和「是否用了 tessellation」。

### 5.3 诊断对照表

| Profiler 现象 | 可能根因（按概率排序） |
|--------------|-------------------|
| `Large Surface Using Direct Render Mode` | ① 顶点着色器里有纹理采样 ② 使用了 tessellation / GS ③ surface 分辨率过大 ④ 顶点/draw 数过少 ⑤ 附件格式过重 |
| `Excessive Binning Pass Time` | ① 图元数量爆炸 ② Position Shader 过复杂（含纹理采样） ③ 大量跨 bin 的超大三角形 ④ 并发分箱被依赖链破坏 ⑤ tile 数过多（分辨率 / MSAA / RT 数） |
| `Texture Fetch Stall` + `% Time ALUs Working` 低 | ① 纹理带宽不足 ② 附件溢出导致主存往返 ③ 纹理未启用压缩（UBWC/AFBC） ④ 采样点不连贯（cache 命中率低） |
| 带宽异常高但代码看起来正常 | ① 附件未设 `LOAD_OP_CLEAR` / `DONT_CARE`，驱动被迫读回 ② `MUTABLE_FORMAT_BIT` 等标志禁用了 UBWC ③ 帧内中途切换 RT |
| 顶点耗时远高于预期 | ① Position Shader 含纹理采样 ② 顶点缓冲交错布局 ③ 索引有空洞导致重复着色 ④ 顶点复用率低（每三角形 > 3 次着色） |

---

## 6. Resolve 成本：被低估的主线成本

### 6.1 什么是 Resolve

**Resolve = 把片上 tile buffer 的最终内容打包写回系统内存。**

```
                 片上 tile buffer
                 ┌─────────────┐
                 │ 32×32 像素   │
                 │ R G B A     │
                 └──────┬──────┘
                        │ 打包 + 总线事务
                        ▼
              系统内存中的帧缓冲（线性布局）
```

### 6.2 为什么它是移动端最重要的成本

⚠️ **每一次「渲染到贴图」都产生一次 Resolve。** Meta/Oculus 官方博客给出的量化（Quest 1 时代）：

> "Because every texture that you render to requires a resolve (back of napkin math is about **half a millisecond for each eye buffer** on Oculus Quest)... you've bumped your resolve cost from **~1ms for forward pass rendering to 3ms+**."

**这意味着**：

| 管线设计 | 需要 Resolve 的 RT 数 | 累计 Resolve 成本（示例环境） |
|---------|---------------------|--------------------------|
| 纯 Forward 一帧到屏幕 | 1（最终帧缓冲） | ~1 ms |
| Deferred（G-Buffer 不回写） | 1（最终颜色） | ~1 ms |
| Deferred + 后处理链（Bloom 多级 + SSAO + DOF + 色调映射） | 5–10 | 3 ms+ |

⚠️ **注意**：即使 G-Buffer 设了 `storeOp = DONT_CARE` 不回写，**每次「渲染到离屏贴图」这个动作本身仍然会有 Resolve**（如果该贴图最终要被后续 pass 当作纹理采样）。这是移动端后处理链必须精简的算术依据。

### 6.3 减少 Resolve 的手段

| 手段 | 说明 |
|------|------|
| **合并 pass**：用 input attachment 把多个 pass 合成一个 render pass | 中间结果不落主存，省掉中间的 Resolve |
| **不要把中间结果当纹理采样** | 用 local read 替代 sampler |
| **Upscale 而非逐级降采样** | 减少后处理链的级数 |
| **省掉不必要的后处理** | 移动端 MO：Bloom 用低分辨率 + 少量 mip；SSAO 可省或用屏幕空间近似的廉价方案 |
| **`STORE_OP_DONT_CARE` / `VK_ATTACHMENT_STORE_OP_NONE_QCOM`** | 明确告知不写回，驱动可完全跳过 Resolve |

⚠️ **特别注意**：`STORE_OP_DONT_CARE` 只在**该附件之后不再被读取**时才有效。如果你设了 DONT_CARE 但后面又采样它，读到的内容是未定义的——这是移动端渲染 artifact 的常见来源。

---

## 7. Load/Store Op 完全指南

### 7.1 决策表

| 附件角色 | `loadOp` | `storeOp` | 理由 |
|---------|---------|----------|------|
| 最终颜色附件（会被 swapchain 呈现） | `CLEAR` 或 `DONT_CARE` | `STORE` | 必须写回 |
| 最终颜色附件 + 后续 pass 叠加（UI） | `LOAD` | `STORE` | 需要保留已有内容 |
| G-Buffer MRT（全屏绘制覆盖） | `DONT_CARE` | `DONT_CARE` | 加载无意义，存储无意义 |
| G-Buffer MRT（部分覆盖，如延迟贴花） | `DONT_CARE` | `DONT_CARE` | 同上，注意保证覆盖面 |
| 深度（仅本 pass 用） | `CLEAR` 或 `DONT_CARE` | `DONT_CARE` | 写完不用 |
| 深度（跨 pass 复用做 early-Z） | `LOAD` | `STORE` | 两个 pass 都要用，尽量合并成一个 pass |
| 阴影图 | `CLEAR` | `STORE` | 后续采样 |
| 后处理中间贴图 | `DONT_CARE` | `DONT_CARE` | 全屏覆盖，之后仅本 pass 内读取 |

### 7.2 Vulkan 代码骨架

```cpp
// ─────────────────────────────────────────────────────────────────────────────
// 移动端 Deferred 单 render pass：几何 → 光照，全部在片上闭环
// ─────────────────────────────────────────────────────────────────────────────

// ① 附件描述：G-Buffer 全 DONT_CARE
VkAttachmentDescription gbufAttachments[2] = {};
for (int i = 0; i < 2; ++i) {
    gbufAttachments[i].format         = kGBufferFormats[i];   // RGBA8 / RGBA8
    gbufAttachments[i].samples        = VK_SAMPLE_COUNT_1_BIT; // ⚠️ G-Buffer 绝不用 MSAA
    gbufAttachments[i].loadOp         = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
    gbufAttachments[i].storeOp        = VK_ATTACHMENT_STORE_OP_DONT_CARE; // ★ 用完即弃
    gbufAttachments[i].stencilLoadOp  = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
    gbufAttachments[i].stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
    gbufAttachments[i].initialLayout  = VK_IMAGE_LAYOUT_UNDEFINED;        // ★ 不关心旧内容
    gbufAttachments[i].finalLayout    = VK_IMAGE_LAYOUT_UNDEFINED;        // ★ 之后不再被读
}

// ② 最终颜色附件
VkAttachmentDescription colorAttachment = {};
colorAttachment.format        = swapchainFormat;
colorAttachment.samples       = VK_SAMPLE_COUNT_1_BIT;
colorAttachment.loadOp        = VK_ATTACHMENT_LOAD_OP_CLEAR;
colorAttachment.storeOp       = VK_ATTACHMENT_STORE_OP_STORE;   // 要呈现
colorAttachment.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
colorAttachment.finalLayout   = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR;

// ③ 深度附件
VkAttachmentDescription depthAttachment = {};
depthAttachment.format        = VK_FORMAT_D32_SFLOAT;
depthAttachment.loadOp        = VK_ATTACHMENT_LOAD_OP_CLEAR;
depthAttachment.storeOp       = VK_ATTACHMENT_STORE_OP_DONT_CARE; // ★ 光照 pass 后不再需要
depthAttachment.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
depthAttachment.finalLayout   = VK_IMAGE_LAYOUT_UNDEFINED;

// ④ Subpass 0：几何 Pass → 写 G-Buffer + 深度
//    Subpass 1：光照 Pass → 读 G-Buffer 作为 input attachment，写最终颜色
VkAttachmentReference gbufRefs[2] = {
    { 0, VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL },
    { 1, VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL },
};
VkAttachmentReference gbufInputRefs[2] = {
    { 0, VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL },
    { 1, VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL },
};
// ... 填写 VkSubpassDescription ...

// ⑤ ⚠️⚠️ 最关键的一步：subpass 依赖必须带 BY_REGION
VkSubpassDependency dep = {};
dep.srcSubpass    = 0;
dep.dstSubpass    = 1;
dep.srcStageMask  = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
dep.srcAccessMask = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT;
dep.dstStageMask  = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;
dep.dstAccessMask = VK_ACCESS_INPUT_ATTACHMENT_READ_BIT;
dep.dependencyFlags = VK_DEPENDENCY_BY_REGION_BIT;  // ★ 没有这一行，驱动会走主存往返！
```

⚠️ **`VK_DEPENDENCY_BY_REGION_BIT` 是移动端 subpass 的生命线**。官方（Qualcomm）原话：

> "Always use `VK_DEPENDENCY_BY_REGION_BIT` for subpass dependencies and pipeline barriers that might execute during per-tile blocks – **omitting `VK_DEPENDENCY_BY_REGION_BIT` will probably deactivate `VK_QCOM_tile_shading` and remove all associated benefits.**"

漏掉它，或者写成 `VK_DEPENDENCY_DEVICE_GROUP_BIT` 之类，就意味着「这个依赖可能跨区域」→ 驱动必须假设数据要落主存。

### 7.3 验证 subpass 是否真的被合并

```cpp
// VK_EXT_subpass_merge_feedback：查询驱动是否把 subpass 合成了同一个 tile pass
VkRenderPassSubpassFeedbackCreateInfoEXT feedbackCI{};
feedbackCI.sType = VK_STRUCTURE_TYPE_RENDER_PASS_SUBPASS_FEEDBACK_CREATE_INFO_EXT;

VkRenderPassCreateInfo2 rpCI{ VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO_2 };
rpCI.pNext = &feedbackCI;
// ... 填写 render pass ...
vkCreateRenderPass2(device, &rpCI, nullptr, &renderPass);

// 对每个 subpass 检查 merge 结果
// feedbackCI.pSubpassFeedback[i].subpassMergeStatus ==
//     VK_SUBPASS_MERGE_STATUS_MERGED_EXT          → ✅ 已合并，数据留在片上
//     VK_SUBPASS_MERGE_STATUS_NOT_MERGED_*        → ❌ 未合并，会走主存
//         （原因包括：依赖不是 BY_REGION、附件冲突、view mask 不匹配等）
```

**这个扩展的价值**：把「subpass 是否真的在片上闭环」从猜测变成**可断言的确定事实**，非常适合接进 CI。

---

## 8. 显式的片上内存管理（Vulkan 上的进阶能力）

### 8.1 相关扩展总览

| 扩展 | 版本 | 作用 | 状态 |
|------|------|------|------|
| `VK_QCOM_tile_properties` | rev 1（2022-07） | **查询** tile 尺寸、apron、origin | Qualcomm |
| `VK_QCOM_tile_shading` | — | 允许在 per-tile block 内执行 draw/dispatch | Qualcomm |
| `VK_QCOM_render_pass_store_ops` | — | 提供 `VK_ATTACHMENT_STORE_OP_NONE_QCOM`（真正的零写回语义） | Qualcomm |
| `VK_QCOM_tile_memory_heap` | rev 1（2025-03） | 允许把 `VkImage` / `VkBuffer` **显式分配**到 tile memory heap 并跨 pass 常驻 | Qualcomm |
| `VK_EXT_subpass_merge_feedback` | — | 查询 subpass 是否被合并 | 跨厂商 |
| `VK_KHR_dynamic_rendering_local_read` | — | dynamic rendering 下的 framebuffer-local 读 | **Vulkan 1.4 核心** |
| `VK_EXT_rasterization_order_attachment_access` / `VK_ARM_...` | — | 无需 barrier 的片上 framebuffer fetch | 跨厂商 |

### 8.2 VK_QCOM_tile_memory_heap：G-Buffer 常驻 GMEM

这是当前 Vulkan 上**让「移动端延迟渲染 G-Buffer 常驻片上」最直接的手段**。

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  传统方式 vs tile_memory_heap 方式                                            │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  传统（隐式）：                                                                │
│    G-Buffer 是普通 VkImage，落在系统内存                                        │
│    → 驱动"尽力"把当前 tile 内容缓存在 GMEM 中                                   │
│    → 生命周期由驱动决定，开发者无法保证                                         │
│    → 一旦遇到多个 pass 就要写回主存                                             │
│                                                                              │
│  tile_memory_heap（显式）：                                                    │
│    G-Buffer 直接分配到 tile memory heap，标 VK_IMAGE_USAGE_TILE_MEMORY_BIT_QCOM│
│    → 语义上"它就在片上"                                                        │
│    → 跨多个 render pass 常驻（在提交批次边界内）                                 │
│    → 内存需求可查询（VkTileMemoryRequirementsQCOM）                             │
│    → 内存不足时开发者可自行决定降级策略（而不是被驱动悄悄 spill）                  │
└──────────────────────────────────────────────────────────────────────────────┘
```

**用法要点**：

```cpp
// ① image 创建时标记 tile memory usage
ici.usage |= VK_IMAGE_USAGE_TILE_MEMORY_BIT_QCOM;

// ② 查询它在 tile memory 中的需求
VkTileMemoryRequirementsQCOM tmReqs{ VK_STRUCTURE_TYPE_TILE_MEMORY_REQUIREMENTS_QCOM };
// → tmReqs.size / tmReqs.alignment
//   ⚠️ 如果 size == 0，说明该格式不支持放入 tile memory，必须走回退路径

// ③ 从 tile memory heap 分配（heap 带 MEMORY_HEAP_TILE_MEMORY_BIT_QCOM）

// ④ 命令缓冲中绑定
vkCmdBindTileMemoryQCOM(cmd, &tileMemoryBindInfo);
//   ⚠️ 必须在 render pass 实例之外调用

// ⑤ 若使用 VK_QCOM_tile_properties，需在 VkRenderingInfo 里声明用量
VkTileMemorySizeInfoQCOM sizeInfo{ VK_STRUCTURE_TYPE_TILE_MEMORY_SIZE_INFO_QCOM };
sizeInfo.size = reservedBytes;   // 必须与实际绑定的 tile memory 字节数一致
```

**限制（务必记住）**：

| 限制 | 说明 |
|------|------|
| resolve attachment **不得**绑定到 tile memory | 硬性禁止 |
| tile memory 内容是**别名**的 | 多个对象绑到同一范围会互相覆盖 |
| 生命周期 | 默认只在**命令缓冲提交批次**边界内保持；是否可扩展到队列提交边界需查询 `queueSubmitBoundary` |
| **不支持的操作** | compute / fragment shader 都不得向 depth/stencil attachment、resolve attachment、input attachment 写入；fragment shader 不得把 color attachment 当 storage image 做 load/store |
| 支持的例外 | **compute shader 向 color attachment 存储是支持的**，可能表现可接受 |
| tile 属性变动态 | `VK_QCOM_tile_memory_heap` 使可用 tile memory 随 render pass 变化 → **tile 属性不再是静态常量** |

### 8.3 Per-Tile Draw 的适用边界

⚠️ `VK_QCOM_tile_shading` 的 per-tile draw **不是通用优化**，官方有明确边界：

| 规则 | 说明 |
|------|------|
| ✅ 适用 | GPU-Driven：per-tile compute 写间接缓冲 → per-tile `vkCmdDrawIndirect` 消费该缓冲 |
| ❌ 不适用 | CPU-Driven 的常规 draw 放进 per-tile block 很可能**性能下降** |
| ⚠️ 关键陷阱 | **per-tile draw 不受 visibility pass 影响** —— 无论该 draw 在 tile 内是否光栅化，都会执行 |
| 必须保证 | 每个 per-tile draw 只包含覆盖当前 tile 的图元 |
| 必须配合 | 同时启用 `VK_QCOM_tile_memory_heap` + `VK_QCOM_tile_shading`；只启用后者不被推荐 |
| 必须用 | `VK_DEPENDENCY_BY_REGION_BIT` |

---

## 9. 主动模式相关的可执行清单

```
□ A. 附件配置
    □ A1. G-Buffer 所有 MRT：loadOp = DONT_CARE，storeOp = DONT_CARE
    □ A2. 深度：本帧后续不再用 → storeOp = DONT_CARE
    □ A3. 最终颜色：loadOp = CLEAR（不要留默认 LOAD）
    □ A4. 确认没有任何附件设了 LOAD 却没有实际需要读取

□ B. 管线结构
    □ B1. 所有 subpass 依赖带 VK_DEPENDENCY_BY_REGION_BIT
    □ B2. 用 VK_EXT_subpass_merge_feedback 验证 subpass 真的被合并
    □ B3. G-Buffer 与光照在同一 render pass，不拆成两个 pass
    □ B4. 帧内不中途切换 framebuffer 去渲染"副产品"，再切回来

□ C. 格式与采样
    □ C1. G-Buffer 无 MSAA
    □ C2. 法线用 Octahedron 编码 + RG16F / RGB10A2
    □ C3. 位置不写入 G-Buffer，用深度重建
    □ C4. 颜色格式优先 R10G10B10A2（Adreno 硬件优化最优）
    □ C5. 确认没有 MUTABLE_FORMAT_BIT 等会禁用显存压缩的标志

□ D. 片元阶段
    □ D1. 避免写 gl_FragDepth
    □ D2. 减少 discard（会强制 late ZS update）
    □ D3. 谨慎使用 framebuffer fetch（会让 shader 不能作为遮挡体）
    □ D4. 若用 VRS，shading rate 控制在 2×2 以内（Mali）

□ E. 并发分箱
    □ E1. 跨 render pass 复用同一深度附件，帧内不重复 clear
    □ E2. 切断 pass 间"顶点着色器依赖前一个 pass 输出"的链路
    □ E3. 若受 VSYNC 限制，在几何密集的 pass 前安排独立工作
```

---

## 10. 小结

| 要点 | 说明 |
|------|------|
| **一个 tile 是「取图元 → 顶点 → 光栅 → 剔除 → 着色 → 混合 → Resolve」的闭环** | 中间阶段全部在片上；Resolve 是该闭环唯一不可避免的大宗主存写 |
| **tile 之间在时间上重叠，不是串行** | 但硬件不是「每核绑一个 tile」，GMEM 是共享资源，同时活跃 tile 数不可查询 |
| **GMEM 容量差异巨大** | Adreno 630 ≈ 1 MB，Adreno X1 = 3 MB（官方），Mali 小核约 16 KB。差 60 倍以上，优化重点完全不同 |
| **GMEM 不是 cache** | 与系统内存层次分离，需显式管理生命周期，这就是 load/store op 的意义 |
| **tile 尺寸不能设置，只能查询** | Adreno 是可变尺寸（驱动按 GMEM + 格式决定），Mali 名义 16×16。想减 tile 数要用官方的 Bin Minimization 手段 |
| **1×1 tile 的灾难在于元数据与调度，不在于 cache line** | tile list 是稀疏的；真正爆炸的是每个 tile 的列表头开销 + 调度粒度 + Resolve 次数 |
| **溢出不表现为「溢出」，表现为驱动的三级妥协** | 缩 bin → spill 到主存 → 整个 surface 走 Direct Mode |
| **Direct Mode 的官方触发条件与容量只有部分关系** | 首要排查：VS 中纹理采样、tessellation/GS、顶点/draw 数量 |
| **Resolve 是被低估的主线成本** | 每个「渲染到离屏贴图」都有一次；后处理链会让它从 ~1ms 涨到 3ms+ |
| **`VK_DEPENDENCY_BY_REGION_BIT` 是移动端 subpass 的生命线** | 漏掉它，片上闭环全部失效。用 `VK_EXT_subpass_merge_feedback` 可以断言验证 |
| **2025 起有了显式的片上内存管理** | `VK_QCOM_tile_memory_heap` 让 G-Buffer 直接分配在 tile memory heap 并跨 pass 常驻 |

---

*上一篇：[02-分箱Binning机制详解](02-分箱Binning机制详解.md) | 下一篇：[04-移动端延迟渲染与Subpass实践](04-移动端延迟渲染与Subpass实践.md)*
