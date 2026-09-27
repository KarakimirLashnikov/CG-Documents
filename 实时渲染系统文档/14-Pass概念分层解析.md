# 实时渲染系统 — Pass 概念分层解析

> 本文档是「实时渲染系统」系列文档的第 **14** 篇。专门解决一个高频且顽固的困惑：**"Pass"在图形渲染里到底是啥？** 本文不再重复"五层表格"本身，而是给你三样真正能消除困惑的工具——**统一心智模型**、**一帧端到端追踪**、**听到 Pass 时的定位树**——并彻底厘清 Pass 与 Pipeline(PSO) / DrawCall / RenderPass / Stage 这些正交概念的关系。
>
> ⚠️ **阅读前置**：本文多次引用本系列 [06-FrameGraph帧图系统]、[07-RDG与FrameGraph概念辨析]、[13-UE5.8的RDG资源与RHI资源]。若你已读，会非常顺；若没读，至少先记住"RDG Pass 是渲染依赖图里的一个逻辑执行节点"。

---

## 1. 你困惑的根本原因（先对齐病根）

绝大多数关于 Pass 的讲解让人更晕，是因为它们**默认你已经在某一层**，然后直接给那层的定义。但 Pass 真正的麻烦在于：

```
┌──────────────────────────────────────────────────────────────┐
│  病根 1：Pass 是「隐喻」，不是「实体类型」                       │
│                                                                │
│  "Pass" 不是一个 C++ 类、不是一个 Vulkan 对象、不是一种资源。   │
│  它只是一个比喻：                                              │
│        "对某个数据域执行一次有明确输入输出的处理趟"            │
│  这个比喻被复用在 5 个抽象层，每层都"合理"地用了 Pass 这个词。  │
├──────────────────────────────────────────────────────────────┤
│  病根 2：不同人说话默认层不同                                   │
│                                                                │
│  引擎程序说 "RDG Pass"      → 第 5 层（图调度节点）            │
│  驱动工程师说 "VkRenderPass" → 第 1 层（API 对象）             │
│  美术/TA 说 "阴影 Pass"      → 第 3 层（帧流程逻辑阶段）       │
│  你听成一锅，自然晕。                                          │
├──────────────────────────────────────────────────────────────┤
│  病根 3：Pass 和一堆正交维度混着说                             │
│                                                                │
│  Pass（在哪渲染 / 哪个阶段）  ←→  Pipeline（怎么渲染）         │
│  Pass（作用域）            ←→  DrawCall（最小提交单元）        │
│  Pass（逻辑趟）            ←→  VkRenderPass（API 对象）        │
│  它们根本不在同一根轴上，却被同一篇文档并列列出。              │
└──────────────────────────────────────────────────────────────┘
```

> **最重要的一句话**：听到 "Pass"，第一反应不是"它是什么类"，而是**"你说的 Pass 是哪一层的？"** 这一问我没问，所有困惑就源于此。

---

## 2. 统一心智模型：Pass = 有界的、命名的、有序的、有 IO 的 GPU 工作域

抛开所有层，给一个**对所有 Pass 都成立**的定义：

> **Pass = 一段被打上边界标记、有名字、在某一执行顺序中排列、对某个数据域做"有输入→处理→有输出"的 GPU 工作集合。**

```
        ┌─────────────────────────────────────┐
        │   Pass: "BasePass"                   │  ← 有名字(Name)
        │                                       │
        │   输入 ──▶ [ 工作集合(Draw/Dispatch) ] ──▶ 输出
        │   几何/材质   (可含多个 DrawCall)        GBuffer
        │                                       │
        │   边界: 从进入作用域到离开作用域        │  ← 有边界(Boundary)
        └─────────────────────────────────────┘
                  ↑ 在帧顺序里占一个位置            ← 有序(Order)
```

**四个不变属性**（任何层的 Pass 都满足）：

| 属性 | 含义 | 例子 |
|------|------|------|
| **边界 Boundary** | 有明确的开始/结束作用域 | `vkCmdBegin/EndRenderPass`、RDG Pass 节点、marker 范围 |
| **名字 Name** | 可被调试器/性能工具识别 | `"ShadowDepthPass"`、`"Lighting"` |
| **顺序 Order** | 在帧的执行序列里占一个位置 | 阴影 Pass 必须在光照 Pass 之前 |
| **IO** | 声明式或隐式地有输入与输出 | 读 GBuffer，写光照结果 |

**四个变化属性**（这就是为什么 Pass 看起来"不一样"）：

| 变化属性 | 取值 | 导致的结果 |
|---------|------|-----------|
| **数据域** | 几何 / 屏幕像素 / 光线 / 附件 / 图节点 | "一趟"遍历的东西不同 |
| **粒度** | 1 个 DrawCall ~ 整帧 | 有的 Pass 就一次 draw，有的含上千 draw |
| **落地形态** | `VkRenderPass` / RDG 节点 / 引擎 marker | 不同层用不同对象承载 |
| **是否含管线切换** | 是 / 否 | 一个 Pass 内可切多个 PSO |

> **记忆锚点**：Pass 是**"作用域 + 顺序 + IO"**，它**不定义管线状态**。记住这点，后面 RenderDoc 的困惑就解了。

---

## 3. 五层框架（作为本文脊梁，不再是重点）

下面这张表你大概率见过，但请把它当**索引**而非答案——它的作用只是让你在听到 Pass 时能立刻归层。

| 层次 | 典型 Pass | 遍历什么数据域 | 为什么叫 Pass | 落地形态 |
|------|----------|---------------|--------------|---------|
| **① API/驱动层** | `VkRenderPass` / Subpass | 一组渲染目标附件 | 对附件的一次绑定、读写、布局转换 | Vulkan API 对象 |
| **② GPU 工作负载层** | 光栅 Pass、计算 Pass | 图元 / 线程组 | 一次图形或计算管线的提交批次 | `vkCmdDraw` / `vkCmdDispatch` |
| **③ 帧流程/架构层** | 阴影/光照/主/全屏 Pass | 场景几何 / 屏幕像素 | 帧渲染中的一个逻辑阶段，完成特定输出 | 引擎逻辑阶段 |
| **④ 算法/技术层** | 光锥 Pass、SSAO Pass | 锥体采样 / 邻居像素 | 一次算法迭代 | 通常由 Compute Dispatch 完成 |
| **⑤ 引擎图/调度层** | UE RDG Pass | 图节点依赖 | 渲染依赖图中的一个可调度执行节点 | `FRDGPass` 节点 |

> **关键认知**：同一名字**跨层**。例如"阴影 Pass"在 ③ 是逻辑阶段，在 ⑤ 可能被实现为 1 个或多个 Raster/Compute 节点，在 ① 可能对应 1 个或多个 `VkRenderPass`。**层级不同，实体不同，但都叫 Pass**——因为它们都满足第 2 节的统一定义。

---

## 4. 关键纠偏：Pass ≠ 这些容易混淆的东西

这是你最可能真正卡住的地方。逐一击破。

### 4.1 Pass ≠ Pipeline（PSO）⚠️（最高频混淆）

```
错误直觉：  "一个 Pass 用一条管线，所以 Pass ≈ Pipeline"
正确事实：

    Pass（作用域："在哪/哪个阶段渲染"）
        ├─ DrawCall 1 → Bind Pipeline A → Draw
        ├─ DrawCall 2 → Bind Pipeline B → Draw   ← 同一个 Pass 内切换管线
        └─ DrawCall 3 → Bind Pipeline C → Draw

    Pipeline（状态："怎么渲染"）
        ├─ 着色器 / 顶点布局 / 混合 / 深度 / 图元拓扑
        └─ 一条 PSO 可以被多个 Pass 复用
```

- **Pass 回答"在哪渲染 / 哪个阶段"**（渲染目标、附件、逻辑作用域）。
- **Pipeline 回答"怎么渲染"**（着色器、顶点输入、混合、深度、拓扑）。
- **一个 Pass 内可以有 N 条不同管线**；**一条管线可以跨多个 Pass 复用**。
- 为什么会被误导：很多文档把"BasePass"又说成"用某个管线"，让你以为 Pass 内含固定管线。其实 BasePass 内有成百上千个不同材质的 PSO。

### 4.2 Pass ≠ DrawCall

| | Pass | DrawCall |
|---|---|---|
| 本质 | **工作集合 / 作用域** | **最小提交单元** |
| 数量关系 | 1 个 Pass 含 **1…N** 个 DrawCall | 多个 DrawCall 聚合成一个 Pass |
| 例子 | BasePass 含上千个 DrawCall | 单个 `vkCmdDraw` / `DrawIndexed` |

⚠️ **特例陷阱**：口语常说"这个 Pass 就一个 draw"——此时 Pass 与单个 DrawCall 重合，**别因此以为它们等价**。重合只是粒度恰好为 1，不是定义相等。

### 4.3 Pass ≠ VkRenderPass（Vulkan 特有）⚠️

`VkRenderPass` 只是**第 ① 层 API 对象**（描述附件、load/store、subpass 依赖）。它与逻辑 Pass 是 **多对多**：

```
多 : 1   一个逻辑 "ShadowPass"（③）可能拆成多个 VkRenderPass（①）
        （每个光源 / 每级 CSM 一个）

1 : 多   多个 RDG 逻辑 Pass（⑤）可能被 RDG 合并成 1 个 VkRenderPass（①）
        （为减少附件 load/store 与布局转换）
```

### 4.4 Pass ≠ 管线内的"阶段(Stage)"

- **Stage**（顶点着色 → 光栅化 → 像素着色）是**单条 Pipeline 内部的子步骤**。
- **Pass** 是**跨 stage 组织的一段完整工作**。
- 别把 "Geometry Pass"（一个逻辑 Pass，写 GBuffer）和 "Geometry Stage/顶点着色阶段" 混为一谈——名字撞车，但完全不是一回事。

### 4.5 一图速查："X 是不是 Pass？"

| 你听到/看到的词 | 它是 Pass 吗 | 它属于哪根轴 |
|----------------|------------|-------------|
| DrawCall | ❌ 否 | 提交单元（Pass 的组成部分） |
| Pipeline / PSO | ❌ 否 | 渲染状态（Pass 内可切换） |
| `VkRenderPass` | ✅ 是（第①层） | API 对象 |
| Subpass | ✅ 是（RenderPass 内子趟） | API 对象内子作用域 |
| 顶点/像素着色 Stage | ❌ 否 | 管线内部阶段 |
| RDG Pass / `FRDGPass` | ✅ 是（第⑤层） | 图调度节点 |
| Compute Dispatch | ❌ 否（但"计算 Pass"是 Pass） | 提交动作（计算 Pass 由它组成） |
| Marker / Event 范围 | ✅ 是（调试作用域） | 调试/逻辑作用域 |

---

## 5. 端到端追踪：一帧延迟渲染里 Pass 的全景

**这一节是解开困惑的关键。** 我们追踪一帧标准延迟渲染，把每个逻辑 Pass 标注到各层，并明确它"对应几个 DrawCall、几个 PSO、是不是一个 VkRenderPass、在 RenderDoc 里长什么样"。

```
帧流程层（③ 逻辑阶段，你写渲染器时这样想）:
  ShadowPass(×CSM级)
     → DepthPrepass
     → BasePass(GBuffer)
     → LightingPass
     → SSAOPass
     → BloomPass
     → ToneMapping(全屏)
     → UIPass
```

| 逻辑 Pass | 第几层 | 含几个 DrawCall | 含几种 PSO | 对应几个 VkRenderPass(①) | RDG 节点(⑤) |
|----------|-------|---------------|-----------|------------------------|------------|
| ShadowPass（含 3 级 CSM） | ③ | 每个级数百个 | 每种物体不同 | 通常 3 个（每级 1 个） | 1 个 RDG Pass（内部循环 draw） |
| DepthPrepass | ③ | 数百~数千 | 不同深度材质 | 1 个 | 1 个 RDG Pass |
| BasePass（GBuffer） | ③ | **上千个** | **每种材质 1 种** | 1 个（或合并） | 1 个 RDG Pass |
| LightingPass | ③ | 1 个全屏 draw（或分块若干） | 1 种（全屏光照 PSO） | 1 个 | 1 个 RDG Pass |
| SSAOPass | ④ | 1~几个 draw | 1~2 种 | 1 个 | 1 个 RDG Pass |
| BloomPass | ③/④ | 多个降采样/升采样 draw | 几种 | 1 个 | 1 个 RDG Pass |
| ToneMapping | ③ | 1 个全屏 draw | 1 种 | 1 个 | 1 个 RDG Pass |
| UIPass | ③ | 若干 | 若干 | 1 个 | 1 个 RDG Pass |

**观察结论（务必记住）**：

1. **逻辑 Pass 与 DrawCall 数量无关**：BasePass 上千 draw，LightingPass 可能 1 个 draw，但它们都是"1 个逻辑 Pass"。
2. **逻辑 Pass 与 PSO 数量无关**：BasePass 内 PSO 成百上千（每材质一种），LightingPass 内 PSO 只有 1 种。
3. **逻辑 Pass(③) 与 VkRenderPass(①) 是多对多**：见 4.3。

### 5.1 用一段自研 FrameGraph 代码看"逻辑 Pass"到底是什么

```cpp
// 自研 FrameGraph：声明逻辑 Pass（参照本系列 [06]）
FrameGraph fg;

// Pass 1：阴影。内部会循环绘制所有投射阴影的物体
fg.AddPass("ShadowPass", PassType::Raster, [&](PassContext& ctx) {
    for (auto& mesh : shadowCasters) {
        ctx.BindPipeline(mesh.GetDepthPSO());   // 不同物体 → 不同 PSO
        ctx.Draw(mesh);                          // 多次 DrawCall
    }
});

// Pass 2：光照。全屏一次 draw，读 GBuffer 写 HDR
fg.AddPass("LightingPass", PassType::Raster, [&](PassContext& ctx) {
    ctx.BindPipeline(lightingPSO);              // 只有 1 种 PSO
    ctx.DrawFullscreenTriangle();               // 只 1 个 DrawCall
});

fg.Compile();   // 编译器：推导 ShadowPass→LightingPass 的依赖与屏障
fg.Execute();
```

```
编译/执行后真实发生的事：

  "ShadowPass"（1 个逻辑节点 ⑤）
     └─▶ 1 个 VkRenderPass ①（或每 CSM 级 1 个）
            ├─ BindPipeline(PSO_石头)  ── Draw
            ├─ BindPipeline(PSO_金属)  ── Draw     ← 同一 Pass 内切 PSO
            ├─ BindPipeline(PSO_树叶)  ── Draw
            └─ ...（数百个 DrawCall，数百种 PSO）

  "LightingPass"（1 个逻辑节点 ⑤）
     └─▶ 1 个 VkRenderPass ①
            └─ BindPipeline(PSO_光照) ── Draw(全屏三角)  ← 1 draw / 1 PSO
```

> **读到这里如果通了，核心困惑就解决了一大半**：`ShadowPass` 这个名字在代码里是 1 个 `AddPass` 调用（第⑤层节点），但它在 GPU 上展开成**数百 DrawCall + 数百 PSO**；而 `LightingPass` 同样是 1 个节点，却只展开成 **1 DrawCall + 1 PSO**。名字一样（都是"Pass"），体量天差地别——因为 Pass 定义的是**作用域与顺序**，不定义里面的 DrawCall/PSO 数量。

---

## 6. RenderDoc 视角：为什么一个 Pass 下有多个不同管线

这是你引用的困惑点，单独讲透。

### 6.1 RenderDoc 里"Pass"的来源有三类

```
RenderDoc 事件浏览器中的 Pass 节点，可能来自：
  (A) Vulkan 原生：vkCmdBeginRenderPass → vkCmdEndRenderPass
  (B) D3D12 原生：BeginRenderPass → EndRenderPass（若引擎用了）
  (C) 引擎/调试标记：RDG_EVENT_SCOPE / SCOPED_DRAW_EVENT /
      BeginEvent / PushMarker 包出的范围
```

这三类**都不描述管线状态**。管线状态在 **PSO / `VkPipeline`** 里。

### 6.2 同一 RenderPass 内切换管线完全合法

```cpp
// Vulkan：同一个 subpass 内频繁切换管线是设计允许的
vkCmdBeginRenderPass(cmd, &rpBegin, VK_SUBPASS_CONTENTS_INLINE);

vkCmdBindPipeline(cmd, GRAPHICS, pipelineA);  vkCmdDraw(cmd, ...);  // 物体A
vkCmdBindPipeline(cmd, GRAPHICS, pipelineB);  vkCmdDraw(cmd, ...);  // 物体B
vkCmdBindPipeline(cmd, GRAPHICS, pipelineC);  vkCmdDraw(cmd, ...);  // 物体C

vkCmdEndRenderPass(cmd);
// 只要 pipelineA/B/C 与当前 RenderPass 的附件格式、subpass 布局兼容，即合法
```

所以你在 RenderDoc 一个 Pass 下看到不同 PSO 的 DrawCall，**是正常现象，不是错误**。

### 6.3 UE RDG 让事情更"混"但更优

- 一个 RDG Pass（⑤）可发出多个 RHI DrawCall，用不同材质 → 不同 PSO。
- RDG 还可能把**多个 Raster Pass 合并成 1 个 Vulkan RenderPass**（①），减少附件 load/store 与布局转换。
- 因此 RenderDoc 里你看到的"一个 Pass"可能是：
  - 一个 RDG 逻辑节点（如 `BasePass` / `ShadowDepthPass`）；
  - 或多个 RDG Pass 合并后的 Vulkan RenderPass。

### 6.4 三大典型例子

| 例子 | 为什么同 Pass 不同 PSO |
|------|---------------------|
| **BasePass** | 所有不透明物体写同一组 GBuffer 附件 → 同一 RenderPass/逻辑 Pass；但材质不同 → 着色器不同 → PSO 不同 |
| **阴影 Pass** | 都输出同一张阴影图；但物体顶点布局/深度偏移/顶点着色器不同 → PSO 不同 |
| **全屏 Pass** | 可能先用**图形管线**画全屏三角，再切**计算管线**做后续；被同一 marker 包住 → 显示在同一 Pass 下 |

### 6.5 如何在 RenderDoc 判别你看到的是哪种 Pass

1. 展开 Pass 节点，看**第一个事件**：
   - 是 `vkCmdBeginRenderPass` → Vulkan 原生 RenderPass（①）
   - 是 `vkCmdBeginDebugUtilsLabelEXT` / `BeginEvent` → 调试标记（③/⑤）
2. 看 `Pipeline State` 面板，对比不同 DrawCall 的 PSO。
3. 看 `API Inspector` 的 RenderPass 定义，确认附件格式与 subpass。
4. 名字像 `BasePass` / `ShadowDepthPass` / `RDG_EVENT_SCOPE` → 逻辑 Pass（⑤）。

> **一句话**：**Pass ≠ Pipeline。** Pass 管"在哪/哪个阶段"，Pipeline 管"怎么渲染"；一个 Pass 内可有多条 Pipeline，一条 Pipeline 可跨多个 Pass 复用。

---

## 7. 定位树：听到 "Pass" 先问四问

下次任何人（包括 AI）说"某某 Pass"，按这棵树定位：

```
听到 "Pass"
   │
   ├─ Q1. 在哪一层？
   │     ├─ API 层?        → 大概率是 VkRenderPass / Subpass（①）
   │     ├─ GPU 负载层?    → 光栅 Pass / 计算 Pass（②）
   │     ├─ 帧流程层?      → 阴影/光照/主/全屏 Pass（③）
   │     ├─ 算法层?        → 光锥/SSAO Pass（④）
   │     └─ 引擎图层?      → UE RDG Pass（⑤）
   │
   ├─ Q2. 遍历什么数据域？
   │     ├─ 几何图元?    → 光栅/阴影/主 Pass
   │     ├─ 屏幕像素?    → 光照/全屏/后处理 Pass
   │     ├─ 光线/锥体?  → 光锥/光追 Pass（④）
   │     ├─ 附件?       → VkRenderPass（①）
   │     └─ 图节点?     → RDG Pass（⑤）
   │
   ├─ Q3. 输出什么？
   │     ├─ 阴影图 / VSM       → 阴影 Pass
   │     ├─ GBuffer            → BasePass
   │     ├─ 光照结果           → LightingPass
   │     ├─ 最终图像           → 全屏/ToneMapping/UI
   │     └─ 资源状态/内存      → VkRenderPass（①）
   │
   └─ Q4. 在 RenderDoc 里它长啥样？
         ├─ vkCmdBeginRenderPass 开头   → Vulkan 原生（①）
         ├─ BeginEvent / marker 开头   → 调试/逻辑作用域（③⑤）
         └─ 名字像 BasePass/Shadow×    → 逻辑 Pass（⑤）
```

能答出 Q1~Q4，你就能精确说出"这个 Pass 是哪一层的、遍历啥、输出啥、调试器里怎么认"——困惑到此终结。

---

## 8. 常见疑问 FAQ（直接解答你可能的疑惑）

**Q：Pass 是一个具体的类吗？**
A：除了 UE RDG 里 `FRDGPass` 是具体类（它仍是"图节点"语义），多数语境下 Pass **不是类**，而是作用域/阶段的叫法。

**Q：一个 Pass 一定包含 DrawCall 吗？**
A：Raster Pass 含 DrawCall；Compute Pass 含 `Dispatch`（不是 DrawCall）；Copy Pass 含拷贝命令。所以"Pass"与"DrawCall"不是包含关系，而是"作用域包含提交动作"。

**Q：Pass 之间会自动插入屏障吗？**
A：在 RDG / FrameGraph（⑤）里**是**——编译器自动推导；裸 Vulkan（①）里**否**——要手动 `vkCmdPipelineBarrier`。这也是为什么用图系统能少写大量同步代码。

**Q：前向渲染里"光照"算一个 Pass 吗？**
A：**不一定**。前向渲染中光照常融入物体着色（每物体一次前向着色 = 1 个 draw，**不是独立 Pass**）；只有把光照单独抽到全屏阶段时才算一个 Pass。这再次说明：是否成"Pass"取决于**架构选择**，不是几何必然。

**Q：Subpass 算 Pass 吗？**
A：算，但它是 `VkRenderPass` **内部的子趟**（①层内），受限于同一 tile 内存，可读前一 subpass 的 tile 数据。它是 Pass 的"子作用域"，不是新层。

**Q：为什么我问的不同 AI 说法不一样？**
A：因为它们默认所在的层不同，且没先跟你对齐"你说的是哪层"。用本文第 7 节的定位树先对齐层，再问细节，答案就不会打架。

---

## 9. 与本系列的关系 & 小结

| 本文概念 | 本系列对应 | 关系 |
|---------|-----------|------|
| 逻辑 Pass（③） | [06] FrameGraph 的 Pass 声明 | 同一抽象：声明式、有 IO、可被图调度 |
| RDG Pass（⑤） | [13] UE5.8 的 `FRDGPass` / `AddPass` | UE 对逻辑 Pass 的工业实现 |
| VkRenderPass（①） | [13] RHI 层 / 底层 API 对象 | 第①层落地形态，与逻辑 Pass 多对多 |
| Pass ≠ Pipeline | [12] 第三层"视图/绑定" | Pipeline/PSO 是"怎么渲染"，Pass 是"在哪渲染" |

**核心结论（背下来这四条就够了）**：

1. **Pass 是隐喻不是实体**：它是"一次有边界、有名字、有序、有 IO 的处理趟"，复用在 5 个抽象层。
2. **Pass 定义作用域与顺序，不定义管线**：所以一个 Pass 内可切多条 PSO，一条 PSO 可跨多个 Pass。
3. **逻辑 Pass 与 DrawCall/PSO 数量无关**：BasePass 上千 draw 上百 PSO，LightingPass 可能 1 draw 1 PSO，都是"1 个 Pass"。
4. **RenderDoc 里 Pass ≠ Pipeline**：RenderDoc 的 Pass 来自原生 RenderPass 或调试标记，都不含管线状态；看到同 Pass 下不同管线是正常而非错误。

> 一句话收尾：**Pass = 在哪渲染 / 哪个阶段的一次处理趟；Pipeline = 怎么渲染；DrawCall = 最小的提交动作。三者正交，别再绑一起想。**

---

*上一篇：[13-UE5.8的RDG资源与RHI资源](13-UE5.8的RDG资源与RHI资源.md) | 下一篇：[15-UERDG屏障与同步机制源码解析](15-UERDG屏障与同步机制源码解析.md)*
