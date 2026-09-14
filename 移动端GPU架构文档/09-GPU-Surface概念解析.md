# GPU Surface 概念解析

> 本文档是「移动端 GPU 架构」系列文档的第 **09** 篇。
> 「Surface」是移动端（尤其 Adreno）优化里出现频率极高、但含义最混乱的术语之一：它同时出现在 **Vulkan API**、**驱动内部决策** 与 **Snapdragon Profiler** 三个层面，且三者并不是一回事。本文从术语消歧出发，给出 Adreno / Qualcomm 语境下 Surface 的精确定义、它在驱动决策与 Profiler 中的位置，以及可观察、可控制的实践手段。

---

## 1. 为什么需要这篇：一个词，三层含义

```
┌────────────────────────────────────────────────────────────────────────┐
│  看到 "surface" 时，先问一句：它说的是哪一层？                            │
├──────────────┬─────────────────────────────────────────────────────────┤
│  ① 通用图形层 │  一块可被 GPU 绘制/写入的 2D 矩形缓冲                     │
│              │  ≈ render target / color attachment / 一张 image         │
├──────────────┼─────────────────────────────────────────────────────────┤
│  ② 呈现层     │  Vulkan 的 VkSurfaceKHR —— 平台窗口的 WSI 抽象            │
│  (WSI)       │  swapchain 图像呈现到它；present mode 决定呈现节奏         │
├──────────────┼─────────────────────────────────────────────────────────┤
│  ③ 驱动/      │  Adreno 语境：一个渲染目标 / render pass 实例              │
│  Profiler 层 │  驱动**逐 surface** 决定 Binned / Direct 模式；            │
│              │  Profiler **逐 surface** 报告指标   ← 本系列主角           │
└──────────────┴─────────────────────────────────────────────────────────┘
```

⚠️ **最常见的错误就是把 ② 和 ③ 当成同一个东西**。它们分属完全不同层次，仅有一个**二阶**关联（通过 VSYNC，见 §3.3）。

---

## 2. 通用含义：Surface = 一块可被绘制的 2D 缓冲

在计算机图形学的传统用法里，**surface 就是"一块能被 GPU 写入的矩形像素数组"**：

| 历史/现代对应物 | 说明 |
|----------------|------|
| DirectDraw 的 **primary surface / offscreen surface** | 早期 2D 图形 API 的直接叫法 |
| 现代 **render target / color attachment** | Vulkan 里 `VkImageView` 作为附件被渲染 |
| **Swapchain image** | 最终要呈现的那张特殊 render target |

这一层的关键词是「**可被写入的画布**」，与任何厂商内部机制无关。理解这一层有助于明白：为什么各家都沿用 "surface" 这个词来指代"渲染目标"。

---

## 3. 呈现层：Vulkan 的 `VkSurfaceKHR`

### 3.1 它是什么

`VkSurfaceKHR` 是 **WSI（Window System Integration）抽象**，代表**操作系统窗口 / 原生显示目标**。它不是一块内存，而是一个"呈现目的地"的句柄：

- 用 `vkCreateXxxSurfaceKHR`（如 `vkCreateAndroidSurfaceKHR`）从平台窗口创建；
- 在其上创建 `VkSwapchainKHR`，换出若干张图像；
- 渲染完成后 `vkQueuePresentKHR` 把图像呈现到这个 surface。

### 3.2 呈现模式（present mode）与 Surface

呈现模式决定**已渲染好的图像以什么节奏提交给窗口**：

| 模式 | 行为 | 是否 VSYNC 受限 |
|------|------|----------------|
| `VK_PRESENT_MODE_FIFO_KHR` | 图像入队，按序在每次垂直消隐呈现（**唯一保证支持**） | ✅ 是 |
| `VK_PRESENT_MODE_FIFO_RELAXED_KHR` | 同 FIFO，但队列空且帧迟到时立即呈现（可撕裂） | ⚠️ 基本是 |
| `VK_PRESENT_MODE_MAILBOX_KHR` | 尽快渲染，新帧在下次 vblank 替换待呈现帧 | ❌ 否 |
| `VK_PRESENT_MODE_IMMEDIATE_KHR` | 就绪即呈现，可能撕裂 | ❌ 否 |

Qualcomm 官方推荐：

> "Use `VK_PRESENT_MODE_FIFO_KHR` and `minImageCount=3` to most efficiently utilize the GPU (`VK_PRESENT_MODE_MAILBOX_KHR` can sometimes help with frame pacing/latency, but often can cost significant battery and generate significant heat for no benefit, as it may renders frames that are not presented to the player)"

### 3.3 与 Adreno 内部 Surface 的关系：只有间接关联

**直接关系：没有。** 呈现模式不会创建、定义或改变任何 Adreno surface——surface 由你的 **render pass / framebuffer** 产生。

**间接关系：有一条，且只通过「VSYNC → 帧节奏」。**

```
Present Mode = FIFO
   → 应用 VSYNC 受限（GPU 提前干完活，帧末空转等 vblank）
   → 新帧开始时管线是空的，首个 surface 的分箱没有上一帧尾巴可藏
   → 叠加「首个 surface 承担全部几何」→ 帧首大串行气泡 → 并发分箱被破坏
```

Qualcomm 官方对并发分箱的原话（`02` 篇已引用）：

> "Another common case that fails to leverage concurrent binning is when the application is VSYNC limited, and **the first surface processes all the geometry**. In such a scenario, try to schedule some independent work before submitting that geometry-heavy render pass so both the independent work and the first surface's geometry binning can happen in parallel."

---

## 4. Adreno / Qualcomm 语境下的 Surface（核心）

### 4.1 精确定义

**一个 Surface ≈ 一个渲染目标 / 一个 render pass 实例**（一组共享同一 framebuffer 附件集合的 draw）。它是驱动做渲染模式决策、Profiler 报告指标的基本单位。

Qualcomm 官方文档中 "surface" 的用法可以佐证：

| 官方表述 | 说明 |
|---------|------|
| "identifies **the mode these surfaces are being rendered with**" | Profiler 逐 surface 报告渲染模式（Binned / Direct） |
| "the driver employing 'Direct' Render Mode **on a surface** that uses a per-tile block" | Direct Mode 是**逐 surface** 的决策 |
| "**the first surface** processes all the geometry" | 帧内第一个 surface（通常是主几何 pass） |
| "LRZ will be disabled until the next **API-surface-clear**" | clear 的作用域是 surface |
| "Subpass merging only applies when **the given surface** is being rendered in binning mode" | subpass 合并的前提是该 surface 走 binning |

📌 **补充**：Qualcomm 官方 Best Practices 里有一个名为 **「Render Surfaces」** 的章节，但它侧重的是**帧末最终呈现的那张 image**——即 sRGB / 像素格式 / 超分 / 带宽优化。这是 "surface" 在官方文档中的**另一处具体用法**（= 最终渲染目标），与本系列关心的"逐 pass 的调度单位"是同一词的两个相关切面，阅读时需结合上下文判断。

### 4.2 为什么 Surface 是 Adreno 的关键单位

| 层面 | Surface 的作用 |
|------|---------------|
| **FlexRender 模式选择** | 驱动**逐 surface** 在 Binned / Binned Direct / Direct 之间选择（启发式不公开） |
| **并发分箱** | 以 surface 为单位重叠：**Surface N+1 的分箱 ‖ Surface N 的渲染** |
| **Subpass 合并** | 只有该 surface 处于 **binning mode** 时，subpass 合并才生效 |
| **GMEM 溢出妥协** | 缩 bin → spill 主存 → 整个 surface 走 Direct Mode（三级妥协） |
| **Profiler 报告** | Rendering Stages 等指标逐 surface 给出模式与耗时 |

### 4.3 什么构成一个 Surface（边界）

- 一个 **render pass instance**（或 dynamic rendering 的一次 scope）通常对应一个 surface；
- **切换 framebuffer / 附件集合** → 产生新的 surface；
- `loadOp` / `storeOp`、clear / invalidate 定义 surface 的**开端与收尾**，直接影响是否发生 GMEMLoad / GMEMStore；
- 因此「**最小化 render pass 数量**」等价于「**最小化 surface 数量**」。

---

## 5. 与相邻概念的边界

| 概念 | 与 Surface 的关系 |
|------|------------------|
| **Render Pass** | 最接近的对应物。一个 render pass instance ≈ 一个 surface |
| **Subpass** | surface **内部**的细分。多个 subpass 合并后仍属同一个 surface（且要求该 surface 走 binning） |
| **Framebuffer / Attachment** | surface 的**内容**。附件集合决定 surface 的格式、MSAA、RT 数 |
| **Swapchain Image** | 一类特殊的 render target。渲染到它时，那个 pass 也是一个 surface |
| **Tile / Bin** | surface **之下**的划分。一个 surface 被切成若干 bin 逐块渲染 |
| **`VkSurfaceKHR`** | **完全不同层**：窗口抽象，与渲染调度无关 |

---

## 6. 实践：如何观察与控制 Surface

### 6.1 观察（Snapdragon Profiler）

| 手段 | 看点 |
|------|------|
| **Rendering Stages**（Trace 模式） | 逐 surface 显示其渲染模式（Binned / Direct）与 subpass 是否合并 |
| **`Large Surface Using Direct Render Mode`** 警告 | 该 surface 退化为 Direct，带宽优化失效（根因见 `03` §5） |
| **Surface 数量** | 被忽视的头号指标——surface 越多，Resolve 与模式切换开销越大 |
| Vulkan Adreno Layer | 未正确合并 subpass 时记录 `VKDBGUTILWARN003` |

### 6.2 控制

**① 减少 surface 数量**

> "Minimize the number of render passes – for example, any time several consecutive passes use the same formatted color buffer, combine them."

- 合并使用同格式颜色缓冲的连续 pass；
- 不用的 depth/stencil 及时禁用；
- 帧内**避免中途切 framebuffer** 去渲染"副产品"。

**② 让 surface 更容易留在 Binned 模式**

Direct Mode 的官方触发条件（启发式不公开，但官方列出）：

> "generally these scenarios trigger direct mode:
> - **High ratio of texture samples in vertex shaders to vertices**
> - **Small number of vertices and/or draws**
> - **Use of tessellation or geometry shaders**"

对应动作：
- 顶点着色器里**杜绝 / 最小化纹理采样**（VS 纹理采样会跑两次：position-only shader 一次、完整 VS 一次，且会触发 Direct Mode）；
- 移动端避免 tessellation / GS；
- 合并小 draw（instancing / indirect draw / GPU-driven）。

**③ 减轻单个 surface 的片上压力（Bin Minimization）**

> "The developer has several options:
> - **reduce frame buffer resolution**
> - **use Variable Rate Shading** (including foveated rendering) to render fewer fragments
> - **use fewer MSAA samples** (in particular, **MSAAx2 is likely to be practically free**)
> - **render to fewer render targets at once**"

**④ 首个 surface 别承载全部几何**

- 若应用 VSYNC 受限且首 surface 吃掉全部几何 → 在其**之前安排独立工作**（或 compute），让分箱与之并行；
- 若 CPU 受限且能接受 1 帧延迟 → 持续提交，直到 CPU 准备提交 N+2 而 GPU 尚未完成 N，促使 N+1 的分箱与 N 的非分箱工作并发。

**⑤ 选对最终 surface 的格式**

- 颜色：**R10G10B10A2**（硬件对此格式优化最优）；需要更高色深且不要 alpha → R11G11B10A0；需要渐进透明 → RGBA16；
- 深度：优先 **D16**（stencil 不需要且精度够）→ **D24_S8**（需 stencil）→ **D32**；
- 均需配合 **UBWC** 压缩；避免 `VK_IMAGE_CREATE_MUTABLE_FORMAT_BIT` 等会禁用 UBWC 的标志。

---

## 7. 常见误区

| 误区 | 纠正 |
|------|------|
| 「`VkSurfaceKHR` 就是驱动里那个 surface」 | ❌ 完全不同层：前者是窗口抽象，后者是渲染目标 / render pass 单位 |
| 「改 present mode 能优化 surface 行为」 | ❌ 只能间接影响（通过 VSYNC → 帧节奏 → 首 surface 分箱） |
| 「surface = tile / bin」 | ❌ tile 是 surface **内部**的空间划分，二者是上下级关系 |
| 「surface 数量无所谓」 | ❌ surface 越多 → Resolve 越多、模式切换越频繁、并发分箱越难 |
| 「Direct Mode 是被 GMEM 容量逼出来的」 | 🟡 官方口径：首要触发条件是 **VS 纹理采样、顶点/draw 数少、tessellation/GS**，与容量只有部分关系 |
| 「subpass 合并一定会生效」 | ❌ **仅当该 surface 处于 binning mode** 时才生效 |

---

## 8. 速查表

| 问题 | 答案 |
|------|------|
| Adreno 的 surface 是什么？ | 一个渲染目标 / render pass 实例，驱动与 Profiler 的基本单位 |
| 逐 surface 的决策有哪些？ | 渲染模式（Binned/Direct）、并发分箱、subpass 合并、GMEM 妥协 |
| 怎么减少 surface？ | 合并连续同格式 pass；避免帧内中途切 framebuffer |
| 怎么让它留在 Binned？ | 消灭 VS 纹理采样；不用 tessellation/GS；合并小 draw；减 RT 数 / MSAA / 分辨率 |
| 首 surface 要注意什么？ | 别让它承载全部几何（VSYNC 受限时会破坏并发分箱） |
| 与 `VkSurfaceKHR` 的关系？ | 无直接关系；仅通过 VSYNC 帧节奏有二阶关联 |

---

## 9. 参考资源

**厂商官方文档（一手）**

- Qualcomm《Adreno GPU on Mobile: Best Practices》（文档编号 80-78185-2）
  - Renderer Architecture → Render Pass / Minimize renderpasses / Swapchain
  - Tile-based Rendering → Bin Minimization / **FlexRender™** / **Concurrent Binning** / Tile Shading Vulkan Extensions
  - **Render Surfaces** → sRGB / Upscaling / Bandwidth Optimization / UBWC
  - Z-Buffer；LRZ, Early-Z and Fast-Z；Vertex and Index Buffers；Shaders
- Qualcomm《Snapdragon Game Toolkit — Game Developer Guide》（同文档集，含 Profiler 与 spec sheets）
- Qualcomm《Snapdragon X Elite Architecture Deep Dive》（FlexRender / GMEM 3 MB）

**本系列交叉引用**

| 篇目 | 关联内容 |
|------|---------|
| `00-总览与架构谱系` | FlexRender 三档、四模式横向对比、**Deferred 三重歧义** |
| `02-分箱Binning机制详解` | 并发分箱机制与三条破坏条件、**Surface N+1 分箱 ‖ Surface N 渲染** |
| `03-逐Tile流水线与GMEM` | GMEM 溢出三级妥协、**Direct Mode 触发条件与诊断对照表** |
| `05-性能诊断与优化清单` | `Large Surface Using Direct Render Mode` 诊断树、Surface 数量指标 |
| `06-VkRenderPass弃用与动态渲染迁移` | subpass 合并与 binning 模式的关系、`VK_QCOM_*` 扩展定位 |

---

*上一篇：[08-前向渲染与多管线DrawCall](08-前向渲染与多管线DrawCall.md) | 返回：[系列索引](index.html)*
