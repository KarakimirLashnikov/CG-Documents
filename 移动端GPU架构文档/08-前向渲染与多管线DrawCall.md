# 移动端 GPU 架构 — 前向渲染：一个 RenderPass 内的多管线 DrawCall

> 本文档是「移动端 GPU 架构文档」系列的第 **8** 篇。前七篇在讲架构时，几乎都是拿「一个 RenderPass、一条管线、若干 draw call」当默认心智模型，于是分箱、GMEM、Tile 生命周期、Resolve 这些结论看起来都是「RenderPass 的属性」。但真实的前向渲染管线不是这样：**同一个 pass 里跑十几条甚至上百条不同管线**（不同材质、不同 blend、不同深度状态、不同 shader 变体）才是常态。
>
> 本篇要回答的问题很具体：**这些架构结论还成不成立？哪些会变？怎么变？**
>
> ⚠️ 先给结论，后面逐条展开：**分箱 / GMEM / Tile 生命周期 / Resolve 完全不变**（它们是 pass 级属性）；**Early-Z、FPK、LRZ、HSR、Fragment Prepass 全部会变**——但它们**从来就不是 pass 级属性**，而是「pass 内 draw call 序列」的属性，前面的文档为了讲清单 draw call 把它们讲简单了；**硬件根本不看你切没切管线，它只看每个 draw 的语义特征**（深度比较方向、是否写 ZS、是否读回附件、RT 写掩码、是否 late-Z、是否混合）。

---

## 1. 一句话定位：把概念分三层

前向渲染里「多管线共存于一个 pass」之所以让人困惑，是因为我们把三个**正交层次**的东西混成了一个「渲染流程」：

```
┌────────────────────────────────────────────────────────────────────────────┐
│                     一个 RenderPass Instance 的三层结构                      │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ① PASS 级（架构层）——决定「数据在哪里」                                    │
│     ├─ 分箱：整个 pass 的几何一次性建成 visibility stream                    │
│     ├─ Tile Memory / GMEM：按 pass 的附件集合一次性分配，尺寸由 bpp 决定      │
│     ├─ Tile 生命周期：= 一个 pass instance                                  │
│     └─ Resolve：pass 结束时每份「要保留的附件」各一次                        │
│     ⚠️ 这一层【完全不关心】pass 里有几条管线、多少个 draw                    │
│                                                                            │
│  ② 序列级（剔除层）——决定「做了多少无用功」                                 │
│     ├─ Early-Z / Late-Z                                                    │
│     ├─ Mali FPK（Forward Pixel Kill）                                       │
│     ├─ Immortalis Fragment Prepass                                          │
│     ├─ Adreno LRZ（Low Resolution Z）                                       │
│     └─ PowerVR / Apple HSR + ISP flush                                      │
│     ⚠️ 这一层【是 draw call 序列的函数】，多管线时差异全部发生在这里          │
│                                                                            │
│  ③ PER-DRAW 语义级——决定「② 还灵不灵」                                      │
│     每个 draw 的几个 bit：                                                   │
│       深度比较方向 | 是否写 ZS | 是否 discard/gl_FragDepth                   │
│       | 是否读回附件 | RT 写掩码是否覆盖前序 | 是否混合 | 是否有副作用         │
│     ⚠️ 硬件看的是这些 bit，【不是】「这个 draw 用了哪条 VkPipeline」          │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

把这张图记住，后面的所有结论都是它的推论。

---

## 2. 不变的那一层：为什么多管线不影响架构结论

### 2.1 分箱的粒度是「整个 pass 的几何」，不是「一个 draw」

这是最容易被误解的一点。以 Adreno 为例，官方文档把 LRZ 描述为 *"draw order independent depth rejection"*：

```
   BINNING PASS（整 pass 一次）                RENDERING PASS（逐 tile）
┌──────────────────────────────────┐      ┌──────────────────────────────┐
│ 所有 draw 的顶点（不管哪条管线）   │      │ for each tile:               │
│   ▼ Position-only VS             │      │   load 附件 → 光栅 → 着色     │
│   ▼ 按屏幕位置分进 bin            │ ───► │   → store/resolve            │
│   ▼ 同时构建 LRZ（低分辨率 Z）     │      │                              │
│   ▼ 输出 Primitive List 到主存     │      │ 片元读取顺序 = 提交顺序       │
└──────────────────────────────────┘      └──────────────────────────────┘
        ▲ 这一层看不到"管线"这个概念                ▲ 这一层才看到逐个 draw
```

**推论 1**：一个 pass 内塞 200 条管线，只付**一次**分箱开销；拆成 5 个 pass，付 5 次分箱 + 5 次 Resolve（系列铁律①）。**所以在移动端，「合并进一个 pass」通常是正确的默认值。**

**推论 2（反直觉）**：Qualcomm 官方给出的 Direct Mode（绕过分箱）触发条件包括「顶点数或 draw 数**少**」。也就是说，**一个 pass 里 draw call 越多，Adreno 反而越不可能走 Direct Mode**——多管线共存相当于把 pass 往 Binned 那边推。这是好消息。

### 2.2 GMEM / Tile Memory 的分配与管线数无关

Tile 的尺寸只由**附件的每像素字节数（bpp）**决定，与你切多少管线、发多少 draw 毫无关系。Arm 在 Fragment Prepass 文档里给了 Immortalis-G925 的实测阈值：

| G-Buffer 总 bpp | G925 上 tile 尺寸 | 后果 |
|---|---|---|
| ≤ 128 bpp | 64 × 64 | 基准 |
| > 128 bpp | 64 × 32 | 跨 tile 图元变多 → 顶点重着色变多 |
| > 256 bpp | 32 × 32 | 更严重的重着色 + 更难隐藏 prepass→main pass 的依赖 |

Arm 的官方建议是 **G-Buffer 尽量 ≤ 128 bpp，最坏不要超过 256 bpp**。注意这条是**附件布局的属性**，不是管线数的属性——100 条管线写同一组 128bpp 附件，tile 尺寸不会变。

⚠️ **但有一个坑**：多材质前向渲染经常出现「某些材质多写一个 RT」（见 §5.3），这会**抬高 bpp 并触发 RT 掩码不兼容**，两条一起把 tile 打小。这是多管线间接影响架构层的唯一常见路径。

### 2.3 Tile 的生命周期仍然是整个 pass instance

切换 `VkPipeline` **不会** flush tile。Tile Memory 的生命周期严格等于 render pass instance（或 dynamic rendering 的一次 `vkCmdBeginRendering` ~ `vkCmdEndRendering`）。

真正会打断片上闭环的是这几样：

| 操作 | 是否破坏片上闭环 | 备注 |
|---|---|---|
| `vkCmdBindPipeline` | ❌ 不会 | 只是改状态，tile 内容保留 |
| `vkCmdBindDescriptorSets` | ❌ 不会 | 同上（Arm：仅 CPU 侧重建内部表） |
| **pass 边界**（End + Begin） | ✅ 会 | 这是 Resolve 的来源 |
| pass 内 `vkCmdClearAttachments` 清深度 | ⚠️ 严重 | 破坏 FPK / LRZ / Fragment Prepass，Arm 明确要求避免 mid-frame depth clear |
| pass 内读自己正在写的附件（feedback loop） | ✅ 会 | 驱动必须 flush；Metal/Apple 官方要求先 snapshot |
| `loadOp = LOAD` 且上一帧内容相关 | ⚠️ 代价 | 每 tile 一次主存读，附件越大越贵 |

> ⚠️ 网上流传「Mali 上频繁 `vkCmdBindPipeline` 会导致 GPU 内部状态刷新」的说法，我只在二手中文/英文博客里见到，**没有找到 Arm 官方原文佐证**。可以确认的 Arm 官方表述是：**描述符集变化会让驱动重建内部描述符表，变化后的第一个 draw CPU 开销更高**（Arm GPU Best Practices, "Optimizing descriptor sets and layouts"）。所以保守结论是：**管线切换主要是 CPU 侧成本，不是 GPU 侧 tile flush**；具体量级请用 Streamline / Snapdragon Profiler 实测，不要照抄任何人的百分比。

---

## 3. 会变的那一层①：可见性剔除从来就是「序列属性」

这是本篇的核心。前三篇讲 Early-Z / HSR / FPK 时，为了讲清楚机制，隐含了「一个 draw 或一组同构 draw」的前提。**一旦 pass 内出现异构的多条管线，这些机制的收益会在 0% 和接近 100% 之间摆动。**

### 3.1 四种机制的对比（这是本篇最重要的一张表）

| 机制 | 厂商 / 引入 | 工作阶段 | 剔除粒度 | **是否依赖 draw 顺序** | 在多管线 pass 中的表现 |
|---|---|---|---|---|---|
| **Early-ZS** | 通用 | 光栅后、着色前 | 逐像素/逐 quad | ✅ **强依赖**：必须从近到远 | 管线交错会打乱顺序 → 收益骤降 |
| **Mali FPK** | Arm，Mali-T620+ | 着色前的 FIFO 队列 | 逐 quad | ⚠️ **部分依赖**：从远到近时靠"后来者 kill 前者" | 能救一部分，但队列深度有限 |
| **Fragment Prepass** | Arm，Mali-G725 / Immortalis-G925+ | 独立 pre-pass | 逐 sample | ❌ **顺序无关** | 最强，但**遇第一个不兼容 draw 即终止** |
| **Adreno LRZ** | Qualcomm，Adreno 6xx 完整支持 | 分箱时构建，渲染时剔除 | LRZ tile（4×4 或 8×8 块） | ❌ **顺序无关** | 与顺序无关，但被"深度方向切换"整段废掉 |
| **PowerVR / Apple HSR** | Imagination / Apple | ISP，光栅后着色前 | 逐像素 | ❌ **顺序无关**（对不透明） | 遇混合/透明要 flush ISP |
| **Depth Prepass** | 软件方案 | 两个 draw 序列 | 逐像素 | — | 见 §6，Arm 明确反对，Apple 有条件建议 |

### 3.2 Early-ZS：唯一强依赖顺序的机制，也是唯一普适的

Arm 官方的原话非常直白：

> *"To get the highest fragment culling rate from the early ZS unit, first render all opaque meshes in a front-to-back render order."*

```
  从近到远（正确）                        从远到近（错误）
┌─────────────────────────┐          ┌─────────────────────────┐
│  ███ 近处墙（先画）      │          │  ░░░ 远处山（先画）      │
│  ┌───────────────────┐  │          │  ┌───────────────────┐  │
│  │ 后续全部被 Early-Z  │  │          │  │ 全画了一遍，再被    │  │
│  │ 挡掉，只着色 1 层   │  │          │  │ 覆盖 → overdraw=N  │  │
│  └───────────────────┘  │          │  └───────────────────┘  │
└─────────────────────────┘          └─────────────────────────┘
     着色像素 ≈ 1.0 × 屏幕                着色像素 ≈ 4.7 × 屏幕（Shadowgun 实测均值）
```

**多管线的破坏方式**：如果按「管线/材质」聚合排序（这对 CPU 友好、对状态切换友好），深度顺序就被打乱了，Early-Z 收益归零。这个冲突是**前向渲染多管线场景最经典的取舍**，见 §5.1。

### 3.3 Mali FPK：跨 draw call 的"事后补救"

FPK 的机制（Mali-T620 起）：所有通过 Early-ZS 的 quad 进入一个 **FIFO 队列**等待着色；如队列中已有 quad B 与 B 同位置、但新进来的 quad A 深度更小，则 **B 被 kill**，不再执行 PS。

```
   FIFO（等待着色）                 新进 A（更近）
   ┌───┬───┬───┬───┐              ┌───┐
   │B1 │B2 │B3 │B4 │   ◄── 比较 ── │ A │ A.depth < B1.depth
   └───┴───┴───┴───┘              └───┘
     └─ B1 与 A 同像素 → B1 被 kill，省一次 PS
```

⚠️ FPK 是**有限深度的补救**，不是 HSR。Arm 官方明确列出 FPK 常见失效场景：

| 失效场景 | 为什么 |
|---|---|
| Alpha 混合的透明 draw | 必须保留顺序语义，不能 kill |
| shader 里程序化访问 framebuffer | 读到的是"kill 前"还是"kill 后"无法保证 |
| **Late-ZS 测试** | 深度在 PS 之后才确定，来不及 kill |
| **小三角形** | quad 尚未填满队列，深度覆盖不足 |

Arm 的官方态度很明确：**"do not rely on the FPK optimization alone. An early ZS test is always more energy-efficient, consistent, and works on older Arm GPUs."**

### 3.4 Immortalis Fragment Prepass：真正的顺序无关，但有"断点"

Mali-G725 / Immortalis-G925 起引入 Fragment Prepass。这是 Arm 对"移动端 HSR"的正式答案：

> *"Fragment Prepass reliably removes overdrawn fragments... As long as there are no incompatible draw calls, Fragment Prepass' culling efficiency is **not sensitive to application draw order**, enabling applications to disable front-to-back sorting of objects in software."*

**这条对多管线前向渲染是巨大的解放**——它意味着你终于可以自由地按管线排序了。但代价是一个硬性限制：

> *"The prepass **terminates at the first incompatible draw in a tile**."*

**不兼容 draw 的官方清单**（逐条对照你的管线）：

| # | 不兼容特征 | 前向渲染里的典型来源 |
|---|---|---|
| 1 | 写 ZS 的**透明** draw | 半透明材质开了 depth write（头发、水面常见） |
| 2 | 透明 draw **之后**的只写 ZS 的 draw | 透明之后的 depth-only pass（如 SSAO 用、或遮挡剔除） |
| 3 | 有副作用、且**读回的值参与颜色计算**的 draw | 读 color attachment 做 cross-lane / subgroup 运算 |
| 4 | **只写前一个 draw 的部分 RT** | ⚠️ 多材质 G-Buffer 的头号杀手，见 §5.3 |
| 5 | 着色受 coverage mask 影响的 draw | centroid 采样、shader 里用 coverage、discard、alpha-to-coverage |

**关键推论**：断点判定是 **per-tile** 的。也就是说，如果一个不兼容 draw 只覆盖屏幕的 1/10 区域，那么只有那部分 tile 的 prepass 被终止，其余 9/10 仍然有效。所以 Arm 的建议是：

> *"Minimize the number of draws that would terminate the prepass, and if unavoidable draw them as late as possible."*

⚠️ 还有一条性能陷阱：Arm 指出 Fragment Prepass 下**简单的 Early-Z draw，顶点着色器里的位置计算最多要跑 3 遍**（分箱一遍、prepass 光栅一遍、main pass 一遍）。所以开启 prepass 后，**VS 的位置计算要尽量精简**，收益比以往更高。

### 3.5 Adreno LRZ：顺序无关，但被"深度方向"一票否决

LRZ 在分箱时构建（低分辨率 Z 缓冲，Mesa 逆向文档确认为 **Z16_UNORM**，粒度 4×4 或 8×8 块），渲染时据此剔除。它是"免费的 depth prepass"。

⚠️ **但它有一条对多管线场景极其致命的限制**：

> *"Since LRZ is an early depth test, such test cannot be used when late-z is required; LRZ buffer could be formed only in one direction, changing depth comparison directions without disabling LRZ would lead to a malformed LRZ buffer."*

也就是说：**只要 pass 内出现了深度比较方向的切换（LESS ↔ GREATER），LRZ 就废了**。而「深度方向」恰恰是**管线状态**——不同材质 pass 用不同 `depthCompareOp`（比如角色用 LESS、天空盒用 LEQUAL、某种 inverted-Z 用 GREATER）是家常便饭。

**精确的规则表**（Igalia 逆向文档 + Mesa freedreno 文档，比官方更细）：

| 变化 | LRZ 状态 |
|---|---|
| `LESS` → `LESS_EQUAL`（或 `GREATER` → `GREATER_EQUAL`） | ✅ 同向，正常 |
| 写深度时 `GREATER` → `LESS` | ❌ **整个 pass 的 LRZ 被禁用** |
| 不同方向但**不写深度** | ⚠️ 仅该 draw 临时禁用 |
| `EQUAL` / `NEVER` | ⚠️ 无方向 → 临时禁用（**你的 stencil 类 draw 不享受 LRZ**） |
| `ALWAYS` / `NOT_EQUAL` | ⚠️ 视是否写深度，临时或完全禁用 |
| shader 写 `gl_FragDepth` | ❌ 完全禁用（除非用 `layout(depth_greater)` 保守深度） |
| shader 里 `discard` | ⚠️ 分箱阶段不能贡献 LRZ，但可在渲染阶段通过 **LRZ feedback** 补回 |
| shader 写 SSBO / image（副作用） | ❌ 强制 Late-Z，与 LRZ 不兼容 |

版本演进（这块很容易搞错，务必按芯片分档）：

| 代次 | LRZ 行为 |
|---|---|
| pre-A650（gen3 前） | 方向在 **CPU** 侧跟踪；方向一变，整个 pass 的 LRZ 失效；**无法在 secondary command buffer 中判断是否启用，也无法跨 render pass 复用** |
| A650+（SD 865+） | 方向在 **GPU** 侧跟踪（`GRAS_LRZ_CNTL.DIR`）；支持**跨 render pass 复用** LRZ（还需 `GRAS_LRZ_DEPTH_VIEW` 匹配） |
| A7XX | **双向 LRZ**：两个方向各一套 buffer，切方向不再禁用（默认关闭，需驱动开启）；另有 4 套 buffer 支持并发分箱 |

⚠️ 注意最后一条的精度问题：**LRZ 用 Z16_UNORM**，Mesa 文档给出 epsilon 为 `1/(1<<16)`，**不足以精确表示 Z32_FLOAT**。所以 inverted-Z / reverse-Z 在 Adreno 上的收益要打问号——你省下的精度可能被 LRZ 的保守化吃掉。

### 3.6 PowerVR / Apple HSR：ISP 的 flush 语义

Apple 与 PowerVR 的做法不同：**ISP 先把一个 tile 的**所有**图元光栅化、做完 HSR，只把"最终可见"的片元交给着色器**。WWDC20 的说法：

> *"HSR will keep track of visibility information. Each pixel will keep the depth and primitive ID for the frontmost primitive."*

这意味着：**对于不透明几何，多管线、多 draw call 完全不影响 HSR 效率——它是顺序无关的，而且不要求你排序。**

但混合（透明）会强制 flush：

```
   HSR 状态：每像素记录 (depth, primitiveID)
   ──────────────────────────────────────────────────────────
   draw 0: 远处蓝三角   → 更新 depth/ID，不着色
   draw 1: 中间橙三角   → 更新 depth/ID，不着色
   draw 2: 前面【半透明】紫三角
            │
            ▼ ⚠️ 需要混合 → HSR 【flush 该图元覆盖的所有像素】
            把已积累的可见片元送进 shader core 着色 → 写回 tile → 再混合
   draw 3: 又一个不透明三角（盖住紫三角）
            → 重新开始积累 depth/ID
```

**推论**：如果透明与不透明**交错**，就会产生"着色 → 马上又被覆盖 → 再着色"的反复 flush。Apple 官方给出的正确顺序是三段式：

```
   ① 不透明（opaque）
   ② 需要 alpha test / discard / 深度反馈的（foliage、树叶、栅栏）
   ③ 半透明（translucent）
```

Apple 的 OpenGL ES 文档还给了两个很有用的补充：
- 用 `discard` 时要**尽早**调用，避免做无用计算；
- **能用 alpha=0 的混合兜底就不要用 discard**（颜色不变，硬件仍可做 Z 优化）；
- 如果 discard 实在无法避免且成为瓶颈，**可以考虑 Z-Prepass**（见 §6 的分歧）。

---

## 4. 会变的那一层②：per-draw 语义位 —— 硬件不看管线，看这些 bit

把所有机制抽象掉，硬件在多管线 pass 里对每个 draw 真正关心的只有这几位：

| 语义位 | 取值 | 影响的机制 |
|---|---|---|
| `depthCompareOp` 方向 | LESS 族 / GREATER 族 / 无方向 | Adreno LRZ（方向切换整段禁用） |
| `depthWriteEnable` | on / off | LRZ、Fragment Prepass（"透明后写 ZS"不兼容） |
| 是否 late-Z | `discard` / `gl_FragDepth` / alpha-to-coverage | Early-Z ❌、LRZ ❌、FPK ❌、Fragment Prepass 需跑部分 FS |
| 是否混合（blend） | on / off | FPK ❌、Apple HSR flush、Fragment Prepass（写 ZS 的透明不兼容） |
| 是否读回附件 | on / off | 片上闭环（feedback loop）、Fragment Prepass、FPK |
| RT 写掩码是否覆盖前序 | 全覆盖 / 部分 | **Fragment Prepass 判定为"透明"**，部分覆盖 + 写 ZS = 不兼容 |
| shader 是否有副作用 | SSBO / image / atomic | 强制 Late-Z，LRZ 与 FPK 双双失效 |

**这张表就是本篇最实用的产出**：做前向渲染的管线矩阵时，按这 7 个 bit 给每条管线打标签，就能预判剔除效率，而不需要去猜"换了管线是不是会变慢"。

⚠️ 一个必须钉死的例子：**"换了管线导致 LRZ 失效"是错误归因**。失效的是**深度方向变了**。同一条管线用 `vkCmdSetDepthCompareOp`（Vulkan 1.3 起可作为动态状态）改方向，一样失效；切 100 条管线但方向一致，一样不失效。

---

## 5. 多管线前向渲染的四个具体陷阱

### 5.1 陷阱一：按管线排序 vs 按深度排序的冲突

这是前向渲染多管线最经典的取舍。PowerVR 官方文档把它讲得非常清楚：

> *"It also prevents the application from sorting draw calls to keep graphics API state changes to a minimum."*

```
   方案 A：严格按深度排序（近→远）        方案 B：严格按管线排序
   ┌──────────────────────────────┐      ┌──────────────────────────────┐
   │ 墙(管线1) 树(管线2) 墙(管线1) │      │ 管线1: 墙 墙 墙 ...           │
   │ 树(管线2) 墙(管线1) 树(管线2) │      │ 管线2: 树 树 树 ...           │
   └──────────────────────────────┘      └──────────────────────────────┘
     GPU 最优：Early-Z 收益最大             CPU 最优：状态切换最少
     CPU 最差：状态切换 N 次                 GPU 最差：overdraw 最大化
```

**分层排序是工程上的标准解**：外层按 pass / 管线族（大粒度，避免频繁切管线），内层按深度桶（小粒度，保留大部分 Early-Z 收益）。

```cpp
// 排序键：外层管线族，内层深度桶。不要追求像素级精确，桶化即可。
struct DrawKey {
    uint32_t passId;      // 位 63..56：render pass / subpass
    uint32_t pipelineId;  // 位 55..40：管线族（不是精确 pipeline！）
    uint32_t depthBucket; // 位 39..24：深度分桶（如 256 桶，非线性分布）
    uint32_t drawId;      // 位 23..0 ：稳定排序
    uint64_t value() const {
        return (uint64_t(passId)     << 56)
             | (uint64_t(pipelineId) << 40)
             | (uint64_t(depthBucket)<< 24)
             |  uint64_t(drawId);
    }
};

// ⚠️ 关键：pipelineId 应该是"管线族"而不是 VkPipeline 的 hash。
// 用 VK_EXT_shader_object 或把 blend/depth 状态做成动态状态后，
// 一族材质只用一条管线，pipelineId 这一层就可以退化为 1~2 个值，
// 此时排序几乎完全由深度决定 —— Early-Z 与低状态切换兼得。
```

⚠️ **在支持 Fragment Prepass（G725/G925+）或 HSR（Apple / PowerVR）的设备上，这个取舍可以直接放弃**——官方明说不需要前到后排序。所以正确的做法是**运行时按 GPU 族切换排序策略**，而不是写死一个。

### 5.2 陷阱二：深度方向在多管线 pass 里悄悄切换

```cpp
// ❌ 反例：一个 pass 内三种深度方向 —— Adreno LRZ 整段报废
pipeline_sky      : depthCompareOp = LESS_EQUAL, depthWrite = false;
pipeline_opaque   : depthCompareOp = LESS,       depthWrite = true;   // 基准方向
pipeline_decal    : depthCompareOp = GREATER,    depthWrite = false;  // ⚠️ 方向反了
pipeline_water    : depthCompareOp = ALWAYS,     depthWrite = true;   // ⚠️ 无方向且写深度

// ✅ 正解 1：统一方向，用 depthBias / 顶点偏移代替方向反转
pipeline_decal    : depthCompareOp = LESS_EQUAL, depthWrite = false;
                    // 用 polygonOffset 或 VS 里沿法线外推解决 z-fighting

// ✅ 正解 2：如果必须反向（如 inverted-Z），把 GREATER 的 draw 拆到独立 pass
//    A650+ 可在 pass 边界复用 LRZ，拆 pass 的代价只有一次 resolve
```

### 5.3 陷阱三：RT 写掩码不覆盖 —— Fragment Prepass 的头号杀手

Arm 官方给的例子值得逐字抄下来：

> *"...if your draw calls do not write to all render targets that have been previously written to, then the draw call is considered transparent. If it then also writes depth or uses stencil, it is considered incompatible!"*

```
   Draw 0（标准材质）: 写 RT0 RT1 RT2 RT3 + depth
   Draw 1（特殊材质）: 写 RT0 RT1 RT2 RT3 RT4 + depth   ← 多写一个 RT4
   Draw 2（标准材质）: 写 RT0 RT1 RT2 RT3 + depth
                       ⚠️ 没有覆盖 Draw 1 写过的 RT4
                       → 被判定为"透明"
                       → 且它写 depth
                       → 【不兼容】→ 该 tile 的 Fragment Prepass 终止
```

**两种解法**：
1. 所有 G-Buffer draw 都写全部 RT（没用的写 0）——**这是 Arm 推荐的做法**；
2. 把"多写一个 RT"的材质拆到独立 pass 或排到最后。

⚠️ 这个坑在 UE / Unity 的自定义材质里极常见：**某个材质多输出一个自定义数据 RT，就把后面所有 tile 的 prepass 全废了**，而现象只是"帧率不明原因掉了 20%"，非常难查。

### 5.4 陷阱四：pass 内清深度

多管线前向渲染里，有时会想在 pass 中间清一次深度（例如分区域渲染、或者某些材质要求干净的 Z）。

> *"Avoid mid-frame depth clears, and avoid late-Z where possible, and failing that make late-Z calculations small."* —— Arm

原因：这会让 FPK 的 FIFO 语义、LRZ 的 buffer 语义、Fragment Prepass 的可见性累积全部失去意义。**要清深度就放到 pass 边界**（`loadOp = CLEAR` 的下一个 pass，或 dynamic rendering 的下一 instance）。

---

## 6. Depth Prepass 在移动端：三种官方立场互相矛盾，别照搬

这是「多管线前向渲染」最常被引用的优化，也是**三方官方口径分歧最大**的一处。

| 来源 | 立场 | 原文要点 |
|---|---|---|
| **Arm（Mali）** | ❌ **明确反对** | *"the cost of the additional draw calls, vertex shading, and memory bandwidth nearly always outweighs the benefits. This is an 'optimization' which actually ends up reducing performance **in all cases we have seen it used**."* |
| **Apple** | ⚠️ **通常不需要，discard 极多时可用** | TBDR 下 HSR 已达成同样目标；但 *"If your performance is limited by unavoidable discard operations, consider a 'Z-Prepass' rendering strategy"*；Apple Silicon Mac 文档直接说 *"this approach is not required anymore because HSR achieves the same goal without additional cost"* |
| **Adreno** | ⚠️ 二手资料称有效 | 有中文技术博客总结"严重过度绘制场景加 depth pre-pass 可提升 20~40% 性能"，但**未找到 Qualcomm 官方原文**，可信度存疑；Adreno 官方文档强调的是用 **LRZ / Early-Z / Fast-Z** 让驱动帮你做，而不是手动 prepass |

**可执行结论**：

```
   是否需要手动 Depth Prepass？
   ────────────────────────────────────────────────────
   你的瓶颈是「片元着色」吗？（profiler 看 fragment cycles / overdraw）
     │
     ├─ 否 → 不要加 prepass。你只是白付一倍 draw + 一倍顶点。
     │
     └─ 是 → 是否大量 discard / alpha-test（植被、栅栏、树叶）？
              │
              ├─ 是 → Apple 平台：官方建议可以做，值得 A/B 测试
              │       Mali：先试排序 + 减少 discard，仍然不行再测
              │       Adreno：先确认 LRZ 是否生效（是不是被方向/副作用废了）
              │
              └─ 否（纯不透明 overdraw）→
                    Mali：不要做（FPK/Prepass 已解决）
                    Apple：不要做（HSR 已解决）
                    Adreno：先排查 LRZ 为什么没生效
```

⚠️ 记住 **Arm 那句话的措辞**：*"in all cases we have seen it used"*。这不是"通常不建议"，这是"我们见过的每一个案例都是负优化"。

---

## 7. 前向渲染 vs 延迟渲染：多管线的分布位置不同

结合本系列第 4 篇，多管线问题在两种架构下的形态完全不同：

| 维度 | 前向渲染（Forward / Forward+） | 延迟渲染（Deferred） |
|---|---|---|
| **管线数量集中在哪** | 主 pass 内：每个材质组合一条管线 | 主 pass 内管线少（G-Buffer 只有几种），**光照 pass 管线多** |
| **一个 pass 内管线数** | **多**（十几 ~ 上百）—— 本篇的主题 | 少（G-Buffer 阶段） |
| **Early-Z / HSR 收益** | 直接：不透明 overdraw 被剔除掉的是**完整光照计算** | 间接：G-Buffer pass 的 overdraw 只是写属性，光照 pass 是每像素一次 |
| **半透明处理** | 同一 pass 后段，天然 | 需要额外 forward pass（UE 的做法），**多一个 pass = 多一次 resolve** |
| **MSAA** | 片上 resolve，相对便宜（系列第 6 / 抗锯齿篇讲过） | 语义冲突（非线性着色不能对输入做线性平均）+ 片上 ×4 |
| **GMEM 压力** | 主 pass 附件少（color + depth） | G-Buffer bpp 高 → tile 变小（128/256 bpp 阈值） |
| **多管线的风险点** | 深度方向、RT 掩码、discard 顺序 | 主要是 bpp 与 tile 尺寸 |

**关键洞察**：**前向渲染把"材质多样性"压进了主 pass 的管线数量里，延迟渲染把它挪到了光照 pass**。所以"多管线"这个问题在前向渲染里是**主战场**，在延迟渲染里不是——但延迟渲染付的是 G-Buffer 带宽与 tile 缩小的代价。**这就是 UE 移动端默认仍是 Forward 的架构根因之一**（04 篇已展开）。

⚠️ **Forward+（Clustered Forward）不改变本篇结论**：它只是把光照列表的计算挪到 compute pass，主 pass 内仍然是一堆材质管线，§3~§5 的所有结论原样适用。

---

## 8. Vulkan 工程清单：一个 pass 多管线的正确写法

### 8.1 硬约束：同一 pass 内所有管线必须兼容

| 约束项 | Vulkan（传统 render pass） | Vulkan 1.4 dynamic rendering |
|---|---|---|
| 附件格式 / sample count | 必须与 `VkRenderPass` 声明一致 | 必须与 `vkCmdBeginRendering` 的 `pColorAttachments` 一致 |
| MSAA 倍数 | 全 pass 统一 | 全 instance 统一 |
| 是否要预先声明管线 | 不需要，但需 `VkGraphicsPipelineCreateInfo::renderPass` 匹配 | ✅ **不需要 `VkRenderPass` 对象**，管线可脱离 pass 创建 |
| subpass 内的 input attachment | 需要在 subpass 描述里声明 | 用 `VK_KHR_dynamic_rendering_local_read` |

⚠️ **dynamic rendering 对多管线场景的真实收益就在这里**：管线不再绑定一个具体的 `VkRenderPass` 句柄，可以跨 pass 复用同一个 `VkPipeline`，组合爆炸被大幅缓解。但**附件格式仍须一致**——想换格式就得换 instance。

### 8.2 减少管线数量：优先用动态状态而不是多管线

```cpp
// ✅ 把易变状态做成动态，一条管线覆盖一族材质
//    Vulkan 1.3 起这些都可以是 dynamic state（核心，无需扩展）
std::vector<VkDynamicState> dyn = {
    VK_DYNAMIC_STATE_VIEWPORT,
    VK_DYNAMIC_STATE_SCISSOR,
    VK_DYNAMIC_STATE_LINE_WIDTH,
    VK_DYNAMIC_STATE_DEPTH_BIAS,
    VK_DYNAMIC_STATE_BLEND_CONSTANTS,
    VK_DYNAMIC_STATE_DEPTH_BOUNDS,
    VK_DYNAMIC_STATE_STENCIL_COMPARE_MASK,
    VK_DYNAMIC_STATE_STENCIL_WRITE_MASK,
    VK_DYNAMIC_STATE_STENCIL_REFERENCE,
    VK_DYNAMIC_STATE_CULL_MODE,                // 1.3
    VK_DYNAMIC_STATE_FRONT_FACE,               // 1.3
    VK_DYNAMIC_STATE_DEPTH_TEST_ENABLE,        // 1.3
    VK_DYNAMIC_STATE_DEPTH_WRITE_ENABLE,       // 1.3
    VK_DYNAMIC_STATE_DEPTH_COMPARE_OP,         // 1.3  ← 注意：见下方警告
    VK_DYNAMIC_STATE_DEPTH_BOUNDS_TEST_ENABLE, // 1.3
    VK_DYNAMIC_STATE_STENCIL_TEST_ENABLE,      // 1.3
    VK_DYNAMIC_STATE_STENCIL_OP,               // 1.3
    // VK_EXT_vertex_input_dynamic_state / VK_EXT_color_write_enable 需扩展
    // VK_EXT_shader_object（1.3 生态）可进一步摆脱"管线对象"本身
};

// ⚠️ 警告：DEPTH_COMPARE_OP 做成动态【不会】让 Adreno LRZ 免于失效。
//    LRZ 看的是实际生效的比较方向，不是"它是不是动态状态"。
//    动态状态省的是【管线数量与切换成本】，换不来【剔除机制的正确性】。
```

其他减少管线数的手段（按优先级）：

| 手段 | 说明 |
|---|---|
| **Bindless**（`VK_EXT_descriptor_indexing`） | 纹理/材质参数走数组索引，一条管线吃下所有材质；Arm 官方建议配 SSBO 而非 UBO |
| **Push Constants / Dynamic Offset** | per-draw 参数不进管线布局 |
| **特化常量**（`VK_SPECIALIZATION`） | 把 `#define` 型变体收敛为同一 SPIR-V 的特化，管线对象仍不同但可共享 `VkPipelineCache` |
| **Pipeline Layout 兼容** | 切换管线时，若新旧 layout 在第 N 集上兼容，该集**不会被 disturb**，无需重绑描述符（Vulkan 规范 "Pipeline Layout Compatibility"） |

### 8.3 推荐的 pass 结构

```
┌────────────────────────────────────────────────────────────────────────┐
│  Pass 0（可选）：Depth / Visibility prepass                              │
│    ⚠️ 仅在 §6 判据成立时使用；Mali 上默认不加                            │
│    loadOp = CLEAR, storeOp = STORE（要被后续采样）                       │
├────────────────────────────────────────────────────────────────────────┤
│  Pass 1：主前向 pass —— 【本篇的主题，多条管线】                          │
│                                                                        │
│    顺序（对照 Apple 官方三段式 + Arm 官方建议）：                         │
│    ┌──────────────────────────────────────────────────────────┐        │
│    │ ① 不透明                                                  │        │
│    │    · 全部 depthWrite = true, 方向统一                      │        │
│    │    · 不支持 prepass/HSR 的设备：从近到远                    │        │
│    │    · 支持 prepass/HSR 的设备：按管线族聚合即可              │        │
│    │ ② alpha-test / discard（树叶、栅栏）                        │        │
│    │    · 尽早 discard；能改 alpha=0 混合就改                    │        │
│    │ ③ 半透明（从远到近）                                        │        │
│    │    · ⚠️ 不要写 depth（否则 Fragment Prepass 判定不兼容）     │        │
│    └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│    附件设置：                                                            │
│      color : loadOp = CLEAR,  storeOp = STORE                          │
│      depth : loadOp = CLEAR,  storeOp = DONT_CARE                      │
│              ⚠️ 移动端：CLEAR 比 LOAD 便宜（系列铁律③）                  │
│      MSAA  : LAZILY_ALLOCATED + TRANSIENT_ATTACHMENT                   │
│              resolve 留在 pass 内，不要单独 resolve                      │
├────────────────────────────────────────────────────────────────────────┤
│  Pass 2+：后处理 / UI（若需要）—— 每个都是一次 resolve，能合则合          │
└────────────────────────────────────────────────────────────────────────┘
```

### 8.4 拆不拆 pass 的判据（多管线版）

| 判据 | 拆 | 合 |
|---|---|---|
| 附件放不进 GMEM / tile 太小（bpp 超 256） | ✅ 必须拆 | |
| 中途要换 depth 比较方向，且是 Adreno pre-A7XX | ✅ 建议拆（A650+ 可跨 pass 复用 LRZ） | |
| 需要读本 pass 正在写的附件（feedback loop） | ✅ 必须拆，或改用 subpass + input attachment | |
| 需要不同 sampleCount / 不同尺寸附件 | ✅ 必须拆 | |
| 只是因为"管线太多想整理一下" | | ❌ **不要拆**：每拆一个 = 一次 resolve |
| 只是因为"两个材质 RT 掩码不同" | ⚠️ 优先改成全写；不行就排到最后，别拆 | |

---

## 9. UE5 / Unity 侧的具体表现

⚠️ 以下引擎相关条目**版本敏感**，以你本地引擎源码为准，尤其是 cvar 取值——二手文档对同一 cvar 常有矛盾说法。

### 9.1 UE5 移动端前向渲染

| 事实 | 说明 |
|---|---|
| 移动端默认 **Forward Shading** | `r.Mobile.ShadingPath`（或项目设置里的 Mobile HDR / Forward）——G-Buffer 路线在移动端不默认 |
| 主 pass 内管线数 | UE 的 `FMobileBasePass` 会按**光照模型 + 材质着色模型 + 混合模式**生成大量 shader 变体，每个变体一条 PSO → 典型场景几十条管线共存于 `MobileBasePass` |
| 半透明 | 独立的 `Translucency` pass（**多一次 resolve**，系列铁律①） |
| 排序 | UE 有自己的排序键（按 depth / 按 state），移动端策略与桌面不同；`r.Mobile.SupportGPUScene`、GPUScene 的 primitive 排序会影响 Early-Z 收益 |
| MSAA | `r.Mobile.AntiAliasing` 取值在二手文档中有矛盾说法（0=off / 1=FXAA / 2=TAA / 3=MSAA 是引擎源码口径），**必须以源码为准**；MSAA 仅 forward 支持，`MobilePPR` 会禁用 MobileMSAA |

**对 UE 的可操作建议**：
- 用 `r.ShaderPipelineCache` / PSO 缓存避免运行时编译卡顿；
- 用 **RenderDoc 抓一帧**，数一下 `MobileBasePass` 里 `vkCmdBindPipeline` 的次数与顺序，对照 §4 的 7 个语义位查有没有"方向切换 / RT 掩码不覆盖"；
- 移动端自定义材质**不要**多输出额外 RT（§5.3）。

### 9.2 Unity URP

| 事实 | 说明 |
|---|---|
| URP 的默认前向路径 | 一个 `RenderPass`（`DrawObjectsPass`）内按 `ShaderPass` + 材质分出多条管线 |
| 排序 | URP 的 `SortingCriteria` 可配（`SortingCriteria.CommonOpaque` 含 front-to-back 的深度排序 + 材质/管线聚合），**两种诉求是编码在同一个排序键里的** |
| SRP Batcher | ⚠️ 它按 **shader 变体**合批，能大幅减少状态切换，但**会与严格深度排序冲突**——URP 的排序会优先保证 batcher 的连续性 |
| 自定义 RenderPass | 用 `ScriptableRenderPass` 插入时，注意 `ConfigureTarget` 的附件选择会决定 resolve 次数 |

**对 Unity 的可操作建议**：
- 在 `UniversalRenderer` 里检查 `SortingCriteria`，明确你是在优化 CPU 还是 GPU；
- 自定义 pass 尽量用 `RenderTargetHandle` 复用已有的 color/depth，别新建 RT。

---

## 10. 误区清单（本篇新增 / 修正）

| 流传说法 | 判定 | 正确表述 |
|---|---|---|
| 「RenderPass 内切管线会 flush tile」 | ❌ 错 | Tile 生命周期 = pass instance。切管线不改它。真正会 flush 的是 pass 边界、pass 内 clear、feedback loop |
| 「多管线会让分箱开销成倍增加」 | ❌ 错 | 分箱是**整 pass 一次**，与管线数无关。反而是拆成多 pass 才会重复付分箱 + resolve |
| 「TBR 下不用管 draw 顺序」 | ⚠️ 部分对 | 只有 **HSR（Apple/PowerVR）、LRZ（Adreno）、Fragment Prepass（G725/G925+）** 是顺序无关的；**Early-Z 强依赖顺序**，且 Arm 官方说 FPK 不可单独依赖 |
| 「Mali 只有 Early-Z，没有 HSR」 | ❌ 错 | 有 FPK（T620+）与 Fragment Prepass（G725/G925+）。系列 01 篇已纠正过一次，本篇再补机制细节 |
| 「移动端也该做 Depth Prepass」 | ⚠️ 看平台 | Arm：**见过的每个案例都是负优化**；Apple：HSR 下不需要，discard 极多时可试；Adreno：先确认 LRZ 生效 |
| 「切管线导致 LRZ 失效」 | ⚠️ 归因错 | 失效的是**深度比较方向改变**，不是"切了管线"。同管线改动态状态一样失效；切 100 条同向管线也不失效 |
| 「每个材质多写一个 RT 没关系」 | ❌ 错 | 触发 Fragment Prepass 的"RT 掩码不覆盖 → 判定透明 → 写 ZS 则不兼容"，会终止该 tile 的 prepass |
| 「`loadOp = LOAD` 比 `CLEAR` 省」 | ❌ 错（移动端） | 系列铁律③：LOAD 意味着每 tile 一次主存读，移动端 CLEAR 更便宜 |
| 「LRZ 能替代深度缓冲精度」 | ❌ 错 | LRZ 是 Z16_UNORM，epsilon `1/65536`，不足以表达 Z32_FLOAT。它是**保守剔除**，不是深度存储 |
| 「多 draw call 更容易触发 Direct Mode」 | ❌ 反了 | Qualcomm 官方触发条件里包含"顶点或 draw 数**少**"。draw 多 → 更可能走 Binned |

---

## 11. 诊断：怎么确认你的多管线 pass 是否健康

| 症状 | 先看 | 工具与指标 |
|---|---|---|
| 帧率高但发热 / 掉帧快 | overdraw | Streamline：`Fragment jobs` / `fragments per pixel` > 1.0 说明有可剔除的 overdraw |
| Late-Z 占比高 | discard / gl_FragDepth / alpha-to-coverage | Arm 明确要求看 *"the number of fragments requiring late-zs testing"* 与被 late-Z kill 的数量 |
| Adreno 上剔除没收益 | 深度方向 / late-Z / 副作用 | Snapdragon Profiler：看 LRZ 相关计数器；对照 §3.5 规则表逐条排查 |
| 掉帧但说不清原因 | RT 掩码不兼容 | 检查是否有材质多写 RT；Fragment Prepass 的断点判定是 per-tile 的，现象会很"局部" |
| CPU 侧 draw 提交慢 | 描述符/管线切换 | Streamline 看 CPU bound；对照 §8.2 合并管线 |
| 顶点是瓶颈 | 顶点重着色 | 跨 tile 的图元会重复跑 varying shading；Prepass 下位置计算最多跑 3 遍 → 精简 VS 位置计算 |

**一条实用口诀**：

> **先看 overdraw（fragments per pixel），再看 late-Z 占比，最后才看管线数量。** 管线数量本身几乎从来不是瓶颈，它只是"序列被切碎"的代理指标。

---

## 12. 小结

| 要点 | 说明 |
|---|---|
| **概念要分三层** | pass 级（架构，不变）→ 序列级（剔除，会变）→ per-draw 语义级（决定剔除是否生效） |
| **架构层完全不受多管线影响** | 分箱、GMEM/tile 分配、tile 生命周期、resolve 都是 pass 的属性 |
| **切管线不 flush tile** | 真正破坏片上闭环的是 pass 边界、pass 内 clear、feedback loop |
| **剔除机制的顺序敏感性各不相同** | Early-Z 强依赖；FPK 部分依赖；LRZ / HSR / Fragment Prepass 顺序无关 |
| **硬件看语义位，不看管线句柄** | 深度方向、写 ZS、discard、读回、RT 掩码、混合、副作用——这 7 位决定一切 |
| **Fragment Prepass 有"断点"** | 遇第一个不兼容 draw 即在该 tile 终止；不兼容 draw 要排到最后 |
| **LRZ 最怕深度方向切换** | 方向一变，整个 pass 的 LRZ 报废（A7XX 双向 LRZ 可缓解，但默认关闭） |
| **Depth Prepass 三方口径不一** | Arm 明确反对、Apple 有条件允许、Adreno 二手资料称有效 → 必须按平台 A/B |
| **合并 pass 是移动端默认值** | 每个额外 pass = 一次 resolve；多管线不是拆 pass 的理由 |
| **多管线是前向渲染的主战场** | 延迟渲染把材质多样性挪到了光照 pass，前向渲染压在主 pass 里 |

---

*上一篇：[07-与主机端的优化差异对照](07-与主机端的优化差异对照.md) | 返回：[系列总览](index.html)*
