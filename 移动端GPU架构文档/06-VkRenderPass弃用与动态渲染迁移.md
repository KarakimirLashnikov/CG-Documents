# 移动端 GPU 架构 — VkRenderPass 弃用与动态渲染迁移

> 本文档是「移动端 GPU 架构」系列文档的第 **6** 篇。本篇厘清 `VkRenderPass` 弃用的**准确版本与含义**，讲明动态渲染在移动端「到底替代了什么、没替代什么」，并纠正关于 `VK_QCOM_tile_shading` 的一处严重误传。

---

## 1. 结论速览：资料判定表

本轮提供的资料在**方向判断**上是对的（动态渲染确实是现代方向、确实与 tile 架构有关），但在**版本事实**和**厂商扩展定位**上有硬伤。

| #  | 原文主张                                                                                    | 判定 | 一句话修正                                                                                                                                                                                                                                            |
| -- | ------------------------------------------------------------------------------------------- | ---- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1  | **「Vulkan 1.3 将 `VkRenderPass` 标记为不推荐」**                                   | ❌   | 弃用发生在**Vulkan 1.4**。1.3 只是把 `VK_KHR_dynamic_rendering` **提升为核心**，`VkRenderPass` 当时仍完全推荐                                                                                                                         |
| 2  | 「1.3 引入动态渲染 → 通过一系列扩展给出 subpass 的答案」                                   | ❌   | 时间线被压缩了。1.3 时代的 dynamic rendering**没有** input attachment / subpass 等价物；补上这块的是 **1.4** 的 `VK_KHR_dynamic_rendering_local_read`                                                                                   |
| 3  | 「启用`VK_QCOM_tile_shading` 会**禁用 FlexRender，强制 GPU 完全运行在 TBDR 模式**」 | ❌❌ | 完全编造。它提供的是「shader 以**tile 为单位** 访问附件」的能力，**不是渲染模式开关**。官方反而要求「尽一切努力**劝阻**驱动在该 surface 上使用 Direct Mode」                                                                        |
| 4  | 把 Adreno 称为 TBDR                                                                         | ❌   | Adreno 是 FlexRender（IMR ⇄ TBR 动态切换），**不是 TBDR**。TBDR 是 Imagination 的术语（含 pixel-perfect HSR）                                                                                                                                  |
| 5  | `VK_EXT_shader_tile_image`「允许片段着色器读取其像素位置上的颜色、深度和模板值」          | ✅   | 描述准确。补充：它**只适用于 dynamic rendering**，定位是 OpenGL `GL_EXT_shader_framebuffer_fetch` 的 Vulkan 等价物（programmable blending）                                                                                                   |
| 6  | 「动态渲染核心思想是把渲染目标指定从**管线创建阶段**提前到命令录制阶段」              | 🟡   | 方向对但对象错。旧模型是在**render pass instance 开始时**通过 `vkCmdBeginRenderPass` 绑定 framebuffer；新模型是在**录制期**用 `VkRenderingInfo` 直接携带附件信息。管线创建阶段仍然要声明附件格式（`VkPipelineRenderingCreateInfo`） |
| 7  | 「1.0 的`VkRenderPass` 初衷是让开发者**显式描述** Tile 局部性」                     | 🟡   | 它提供的是让**驱动可推断** framebuffer-local 依赖的信息，不是让开发者直接描述 tile。开发者从未能描述 tile                                                                                                                                       |
| 8  | 「桌面 IMR 与移动 TBR 对渲染通道理解根本不同，Vulkan 要同时服务两者」                       | 🟡   | 这是 API 复杂度的次要来源。Khronos 给出的主要理由是**「setup 极其繁琐」**本身                                                                                                                                                                         |
| 9  | 「动态渲染更彻底地适配移动端，是减法式优化」                                                | ✅   | 结论正确。但有一个必须补上的限定：**local read 的片上闭环同样依赖 `VK_DEPENDENCY_BY_REGION_BIT`**，它不是自动的                                                                                                                               |
| 10 | 「动态渲染 + Local Read 完整保留了 Tile 架构支持」                                          | 🟠   | 「**大部分**」而非「完整」。官方明确保留了两处缺口：未复刻的 subpass 功能需要拆成多个 render pass instance；`VK_QCOM_render_pass_shader_resolve` **没有**等价物                                                                         |
| 11 | 「你现在遇到的 Direct Mode 警告，说明正确配置这些渲染路径的重要性」                         | 🟡   | 因果链太短。Direct Mode 的首要触发条件（官方口径）是 VS 中纹理采样、tessellation/GS，与渲染路径配置只有部分关系                                                                                                                                       |

**一句话总结**：资料的**叙事方向对，时间线与扩展定位错**。特别是第 3 条，把 `VK_QCOM_tile_shading` 描述成一个「强制 TBDR 模式的开关」，这在官方文档里找不到任何依据，且与其真实用途相反。

---

## 2. 准确的时间线

这是本篇最重要的一节。Khronos 官方 deprecation 附录给出了权威版本：

```
┌────────────────────────────────────────────────────────────────────────────────┐
│  Vulkan 渲染通道 API 的三段式演进                                                │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  【阶段 0】Vulkan 1.0（2016）                                                  │
│    · 引入 `VkRenderPass` + `VkFramebuffer`                                     │
│    · 引入 subpass + input attachment → framebuffer-local 依赖可表达             │
│    · 状态：**推荐使用**                                                        │
│                                                                                │
│  【阶段 1】Vulkan 1.2（2020）                                                  │
│    · `VK_KHR_create_renderpass2` 提升为核心 → 引入 `vkCreateRenderPass2` 等     │
│    · 老的 `vkCreateRenderPass` 等被标记为 deprecated（"deprecation via version  │
│      2"，属技术性替换，功能等价）                                                │
│    · `VkRenderPass` **对象本身仍完全推荐使用**                                  │
│                                                                                │
│  【阶段 2】Vulkan 1.3（2022）                                                  │
│    · `VK_KHR_dynamic_rendering` 提升为核心                                     │
│    · 新增 `vkCmdBeginRendering` / `vkCmdEndRendering`                          │
│    · 不再需要创建 `VkRenderPass` 和 `VkFramebuffer`                            │
│    ⚠️ 关键：**但是 subpass 功能没有等价物**                                     │
│       → dynamic rendering 当时只适合「不用 subpass」的内容                      │
│       → `VkRenderPass` **没有被弃用**，仍是 tile 优化的唯一正道                  │
│                                                                                │
│  【阶段 3】Vulkan 1.4（2024）                                                  │
│    · `VK_KHR_dynamic_rendering_local_read` 提升为核心                          │
│    · 补上 subpass 的主要能力（input attachment 的 local read）                  │
│    · ★★ **此时** `VkRenderPass` / `VkFramebuffer` 才被正式标记为 deprecated    │
│    · Khronos 建议：此后使用 `vkCmdBeginRendering` / `vkCmdEndRendering`        │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
```

### 2.1 官方规范原文（关键两段）

**关于 1.3 的局限**：

> "In **Vulkan 1.3**, the `VK_KHR_dynamic_rendering` extension was promoted into core, which added a new way to specify render passes without needing to create `VkFramebuffer` and `VkRenderPass` objects. **However, subpass functionality had no equivalent, meaning dynamic rendering was only suitable as a substitute for content not using subpasses.**"

**关于 1.4 的补完与残留缺口**：

> "In **Vulkan 1.4** however, `VK_KHR_dynamic_rendering_local_read` was promoted into core as well, which allows the expression of **most** subpass functionality in core or extensions. Any subpass functionality which was not replicated is still expressible but requires applications to **split work over multiple dynamic render pass instances**. Functionality not covered with local reads would result in most or all vendors splitting the subpass internally.
>
> **`VK_QCOM_render_pass_shader_resolve` does not have equivalent functionality exposed via dynamic rendering.** Use of deprecated functionality will be required to use that extension unless/until replacements are created.
>
> Outside of vendor extensions, applications are advised to make use of `vkCmdBeginRendering` and `vkCmdEndRendering` to manage render passes from this API version onward."

⚠️ **注意最后那段里的例外**：`VK_QCOM_render_pass_shader_resolve`（shader resolve，允许在片元着色器里执行 MSAA resolve）**在 dynamic rendering 下没有等价物**。也就是说，「RenderPass 完全被动态渲染替代」这句话有一个 Qualcomm 专有的反例。资料说「完整保留」是不准确的。

### 2.2 为什么 1.3 的 dynamic rendering 不能用于延迟渲染

```
┌────────────────────────────────────────────────────────────────────────────────┐
│  1.3 时代的困境                                                                │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  你要做移动端延迟渲染（几何 Pass → 光照 Pass，片上闭环）：                        │
│                                                                                │
│  ✅ 用 render pass + 2 个 subpass：                                            │
│       subpass 0 写 G-Buffer（color attachment）                                │
│       subpass 1 读 G-Buffer（input attachment）                                │
│       → 依赖是 framebuffer-local，数据留在片上 ✅                               │
│                                                                                │
│  ❌ 用 1.3 的 dynamic rendering：                                               │
│       没有 input attachment，没有 subpass 依赖                                 │
│       → 只能把 G-Buffer 当普通纹理，用 sampler 采样                             │
│       → 必须拆成两个 render pass instance                                     │
│       → G-Buffer 落主存，带宽翻倍 ❌                                            │
│                                                                                │
│  → 所以在 1.3 时代，延迟渲染**必须**用 render pass。                             │
│    这就是为什么 1.3 不可能弃用 VkRenderPass。                                   │
└────────────────────────────────────────────────────────────────────────────────┘
```

Vulkanised 2025 官方幻灯片对此表述得非常直白：

> "Vulkan 1.3 promoted dynamic rendering to core... Greatly simplified the programming model... **but the original extension didn't address input attachments or subpasses • Critical for performance on tile-based GPUs**"

也就是说：**Khronos 自己承认，1.3 的 dynamic rendering 恰好缺的是 tile-based GPU 最需要的那一块。** 补上它的是 1.4。

---

## 3. `deprecated` 的准确含义

### 3.1 它不等于「失效」也不等于「不能用」

Khronos 官方 deprecation 附录原文：

> "Deprecation is tagged in the xml registry... **Deprecated functionality will remain available for backwards compatibility until a new major core version removes it**, but applications targeting more recent versions of Vulkan should still avoid using deprecated functionality."

```
┌────────────────────────────────────────────────────────────────────────────────┐
│  deprecated 的法律含义                                                          │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  ✅ 仍然完全有效                                                               │
│     · 现有的 render pass 代码不会失效                                          │
│     · 验证层不会报错                                                           │
│     · 驱动必须继续支持                                                         │
│                                                                                │
│  ⚠️ 但有三条风险                                                               │
│     ① "Interactions with deprecated functionality will often be omitted when   │
│        new extensions or features are developed"                              │
│        → 新扩展可能**不定义**与已弃用功能的交互行为                              │
│        例如：VK_QCOM_tile_memory_heap 的 tile 属性查询走的是                    │
│              VkRenderingInfo / VkRenderPassCreateInfo 两条路，但新功能          │
│              的交互细节往往只在 dynamic rendering 路径上完整定义                 │
│                                                                                │
│     ② "may not work with the latest features"                                 │
│        → 长期看会成为新特性的盲区                                               │
│                                                                                │
│     ③ 工具链可能开始告警                                                        │
│        → Khronos 已在 validation layer 中提供可选的 deprecated 入口点报告       │
│        → WG 还在讨论「opt-in 编译器警告」或「只含推荐功能的头文件」               │
│                                                                                │
│  ❌ 不会：在 1.x 系列内被移除。移除只发生在下一个 major core 版本。               │
└────────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 完整的弃用清单（渲染相关）

| 被弃用项                                                       | 弃用版本                 | 替代物                                              | 备注                 |
| -------------------------------------------------------------- | ------------------------ | --------------------------------------------------- | -------------------- |
| `vkCreateRenderPass` / `vkCmdBeginRenderPass` (1.0 版)     | 1.2                      | `vkCreateRenderPass2` / `vkCmdBeginRenderPass2` | 技术性替换，功能等价 |
| **`VkRenderPass` / `VkFramebuffer` 对象**            | **1.4**            | **Dynamic Rendering**                         | 本次讨论的对象       |
| `VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT` / `BOTTOM_OF_PIPE_BIT` | 1.4                      | Sync2 的`NONE` / `ALL_COMMANDS`                 | —                   |
| `VkShaderModule`                                             | `VK_KHR_maintenance5`  | Shader Object 等                                    | —                   |
| Monolithic`VkPipeline`                                       | `VK_EXT_shader_object` | Shader Object                                       | —                   |

### 3.3 一个常被忽略的例外

```
┌────────────────────────────────────────────────────────────────────────────────┐
│  ⚠️ VK_QCOM_render_pass_shader_resolve 没有 dynamic rendering 等价物            │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  规范原文：                                                                     │
│    "VK_QCOM_render_pass_shader_resolve does not have equivalent functionality   │
│     exposed via dynamic rendering. **Use of deprecated functionality will be    │
│     required to use that extension** unless/until replacements are created."   │
│                                                                                │
│  含义：                                                                        │
│    · 如果你想用 shader resolve（在片元着色器里做 MSAA resolve）                  │
│    · 就被迫继续使用 render pass 路径                                            │
│    · 这在移动端反而可惜——因为 MSAA 的片上 resolve 恰是 TBDR 的优势              │
│                                                                                │
│  → 结论：不要盲目认为「render pass 路径已经死了」。                             │
│    在你决定迁移前，先确认没有依赖任何没有等价物的厂商扩展。                        │
└────────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. 为什么 `VkRenderPass` 会被取代

### 4.1 Khronos 给出的理由

| 理由                             | 具体表现                                                                                                                                                                |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **setup 极其繁琐**         | Khronos 博客原话：「A big criticism with renderpasses was how involved esp. the setup is. Getting renderpasses and subpasses incl. dependencies correct can be tricky」 |
| **难以融入动态变化的管线** | 「renderpasses are kinda hard to integrate into a dynamically changing setup, making them a**hard fit for complex Vulkan projects like game engines**」           |
| **对象的僵化**             | 附件格式、load/store op、依赖、view mask 任何一处变化都要重建整个 render pass 对象；而现代管线（尤其帧图系统）需要频繁调整                                              |
| **组合爆炸**               | 后处理链的每个环节配置不同 → render pass 对象数量指数增长                                                                                                              |
| **两个对象耦合**           | `VkRenderPass` 定义结构、`VkFramebuffer` 绑定具体 image view，二者必须一致，增减附件要同时改两处                                                                    |

### 4.2 但它的设计初衷确实与移动端有关

这一点资料说得对。`VkRenderPass` + subpass 的设计动机之一，就是给驱动提供足够信息去推断 **framebuffer-local 依赖**，从而使 tile-based 架构能把中间结果留在片上。

Vulkan 规范的措辞：

> "By describing a complete set of subpasses in advance, render passes provide the implementation an opportunity to optimize the storage and transfer of attachment data between subpasses. In practice, this means that **subpasses with a simple framebuffer-space dependency may be merged into a single tiled rendering pass, keeping the attachment data on-chip** for the duration of a render pass instance."

⚠️ 但要注意措辞里的「**may** be merged」和「**an opportunity**」——**合并是驱动的自由裁量，不是保证**。这正是 `VK_EXT_subpass_merge_feedback` 存在的理由（让开发者能验证）。

**所以准确的表述是**：`VkRenderPass` 是「为移动端提供的**提示机制**」，而不是「对移动端 tile 的直接控制」。它同时服务桌面与移动，代价是 API 复杂度。1.4 的替代方案把这份提示能力**保留**（local read 同样带 BY_REGION 语义），同时大幅降低了对象管理成本。

---

## 5. 动态渲染的能力对照

### 5.1 能力矩阵

| 能力                                                     | Render Pass + Subpass                        | 1.3 Dynamic Rendering                      | 1.4 Dynamic Rendering + Local Read             |
| -------------------------------------------------------- | -------------------------------------------- | ------------------------------------------ | ---------------------------------------------- |
| 开始/结束渲染                                            | `vkCmdBeginRenderPass` / `EndRenderPass` | `vkCmdBeginRendering` / `EndRendering` | 同左                                           |
| 附件指定方式                                             | `VkFramebuffer`（预创建对象）              | `VkRenderingAttachmentInfo`（栈上）      | 同左                                           |
| Need`VkFramebuffer` object                             | ✅ 需要                                      | ❌ 不需要                                  | ❌ 不需要                                      |
| Need`VkRenderPass` object                              | ✅ 需要                                      | ❌ 不需要                                  | ❌ 不需要                                      |
| 管线声明附件格式                                         | 引用`VkRenderPass`                         | `VkPipelineRenderingCreateInfo`          | 同左                                           |
| **Framebuffer-local 读（input attachment）**       | ✅                                           | ❌**无**                             | ✅ 有                                          |
| 进阶 subpass 的 subpass 依赖                             | ✅                                           | ❌                                         | ✅ 用 BY_REGION barrier 表达                   |
| 布局                                                     | `SHADER_READ_ONLY_OPTIMAL` 等              | —                                         | `VK_IMAGE_LAYOUT_RENDERING_LOCAL_READ_KHR`   |
| Shader 接口                                              | `subpassInput` / `subpassLoad`           | —                                         | **完全相同**（无需改 shader）            |
| 附件索引重映射                                           | subpass 边界天然切换                         | —                                         | `vkCmdSetRenderingInputAttachmentIndicesKHR` |
| Color 附件定位重映射                                     | subpass 边界天然切换                         | —                                         | `vkCmdSetRenderingAttachmentLocationsKHR`    |
| depth/stencil/MSAA 的 local read                         | ✅                                           | ❌                                         | ⚠️**可选特性**                         |
| shader resolve（`VK_QCOM_render_pass_shader_resolve`） | ✅                                           | ❌                                         | ❌**无等价物**                           |
| 未复刻的 subpass 功能                                    | ✅                                           | ❌                                         | ⚠️ 需拆成多个 render pass instance           |

⚠️ **注意「depth/stencil/MSAA 的 local read 是可选特性」**。Vulkanised 2025 官方幻灯片原文：

> "Vulkan 1.4 includes local reads for **color attachments / storage resources**. Closes the gap versus legacy render passes. **Local reads for depth/stencil/multisampled attachments are optional.**"

**实践含义**：如果你的延迟渲染需要在片元着色器里**读深度附件**，这个能力是**可能不提供**的。稳妥做法仍是第 4 篇建议的：把线性深度写进 G-Buffer 通道，而不是依赖读深度附件。

### 5.2 迁移的对象映射表

| 旧概念                                                   | 新对应                                                 |
| -------------------------------------------------------- | ------------------------------------------------------ |
| `VkRenderPass`                                         | 无（管线用`VkPipelineRenderingCreateInfo` 声明格式） |
| `VkFramebuffer`                                        | 无（附件直接来自`VkRenderingAttachmentInfo`）        |
| `VkRenderPassBeginInfo`                                | `VkRenderingInfo`                                    |
| `vkCmdBeginRenderPass`                                 | `vkCmdBeginRendering`                                |
| `vkCmdNextSubpass`                                     | 带`eByRegion` 的 `vkCmdPipelineBarrier2`           |
| `VkAttachmentReference`（input）                       | `VkRenderingInputAttachmentIndexInfoKHR`             |
| `VkSubpassDependency`                                  | `VkDependencyInfo`（带 `eByRegion`）               |
| `VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL`（input 用） | `VK_IMAGE_LAYOUT_RENDERING_LOCAL_READ_KHR`           |
| `VkSubpassDescription`                                 | 无（按 draw 顺序 + barrier 分隔）                      |

---

## 6. Local Read 的完整机制

### 6.1 三个必要条件

```
┌────────────────────────────────────────────────────────────────────────────────┐
│  local read 生效，必须同时满足三个条件                                           │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  ① 启用特性                                                                    │
│     VkPhysicalDeviceDynamicRenderingLocalReadFeaturesKHR                       │
│       .dynamicRenderingLocalRead = VK_TRUE                                     │
│                                                                                │
│  ② 附件使用专用布局                                                            │
│     VK_IMAGE_LAYOUT_RENDERING_LOCAL_READ_KHR                                   │
│     （layout 值 1000232000）                                                   │
│     可用于：storage image、render pass 的 color / depth-stencil / input        │
│     attachment                                                                 │
│                                                                                │
│  ③ ★★★ barrier 必须带 VK_DEPENDENCY_BY_REGION_BIT                             │
│     且 source / destination stage 都必须是 framebuffer-space 的                 │
│     → 表示「依赖只在同一像素位置内成立」                                          │
│     → 驱动才敢把数据留在片上                                                    │
│                                                                                │
│  缺任何一条 → 退化为跨主存的普通读                                                │
└────────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 barrier 的硬性约束

规范对 local read barrier 的限制（原文归纳）：

| 约束                                                    | 说明                                                                                                                           |
| ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| 必须带`VK_DEPENDENCY_BY_REGION_BIT`                   | 否则不构成 framebuffer-local 依赖                                                                                              |
| stage 必须是 framebuffer-space                          | fragment shader、color attachment output、early/late fragment tests 等                                                         |
| **不能做 layout transition**                      | barrier 中的`oldLayout` 与 `newLayout` 必须相同（都用 `RENDERING_LOCAL_READ`）                                           |
| **不能做 queue family transfer**                  | 不涉及 queue ownership 转移                                                                                                    |
| 读取范围限于「前一个片元着色器写入的值」                | 「**Reading data outside of values written by a previous fragment shader is undefined behavior**」                       |
| storage resource 的语义按**片元位置**而非资源位置 | 规范举例：位置 (5,5) 的片元写了 storage image 的 (6,6) 和 (21,700)，则后续位置 (5,5) 的片元可以读这两处                        |
| 附件写 → 只能通过 input attachment 读                  | 「Writes to attachments can only be made visible in this way via input attachments」                                           |
| 不能用「A 类型写、B 类型读」                            | 「there is still no way to write through one type of resource and then read through another in the same render pass instance」 |

### 6.3 代码骨架（含索引重映射）

```cpp
// ═════════════════════════════════════════════════════════════════════════════
// 从 subpass 迁移到 dynamic rendering + local read
// ═════════════════════════════════════════════════════════════════════════════
namespace {

// 两个 G-Buffer 附件都用 RENDERING_LOCAL_READ 布局
vk::RenderingAttachmentInfo makeLocalReadAttachment(vk::ImageView view) {
    return vk::RenderingAttachmentInfo{}
        .setImageView(view)
        .setImageLayout(vk::ImageLayout::eRenderingLocalRead)  // ★ 专用布局
        .setLoadOp(vk::AttachmentLoadOp::eDontCare)            // ★ 不读旧内容
        .setStoreOp(vk::AttachmentStoreOp::eDontCare);         // ★ 不写回主存
}

}  // namespace

void DeferredRenderer::record(cmd) {
    // ── Pass 1：几何 ────────────────────────────────────────────────────────
    {
        std::array<vk::RenderingAttachmentInfo, 2> gbuf{
            makeLocalReadAttachment(m_gbufView[0]),
            makeLocalReadAttachment(m_gbufView[1]),
        };
        vk::RenderingAttachmentInfo depth{}, finalColor{};
        depth.setImageView(m_depthView)
             .setImageLayout(vk::ImageLayout::eDepthAttachmentOptimal)
             .setLoadOp(vk::AttachmentLoadOp::eClear)
             .setStoreOp(vk::AttachmentStoreOp::eDontCare);

        cmd.beginRendering(vk::RenderingInfo{}
            .setRenderArea(m_fullArea)
            .setLayerCount(1)
            .setColorAttachments(gbuf)
            .setPDepthAttachment(&depth));
        drawOpaqueGeometry(cmd);
        cmd.endRendering();
    }

    // ── ★ Framebuffer-local barrier ─────────────────────────────────────────
    {
        std::array<vk::ImageMemoryBarrier2, 2> barriers{};
        for (size_t i = 0; i < 2; ++i) {
            barriers[i]
                .setSrcStageMask(vk::PipelineStageFlagBits2::eColorAttachmentOutput)
                .setSrcAccessMask(vk::AccessFlagBits2::eColorAttachmentWrite)
                .setDstStageMask(vk::PipelineStageFlagBits2::eFragmentShader)
                .setDstAccessMask(vk::AccessFlagBits2::eInputAttachmentRead)
                .setOldLayout(vk::ImageLayout::eRenderingLocalRead)
                .setNewLayout(vk::ImageLayout::eRenderingLocalRead)  // ⚠️ 相同！
                .setImage(m_gbufImage[i])
                .setSubresourceRange(m_fullRange);
        }
        cmd.pipelineBarrier2(vk::DependencyInfo{}
            .setDependencyFlags(vk::DependencyFlagBits::eByRegion)   // ★★★ 生命线
            .setImageMemoryBarriers(barriers));
    }

    // ── Pass 2：光照（读 G-Buffer，写最终颜色）──────────────────────────────
    {
        // ★ 附件定位重映射：把两个 G-Buffer 附件映射为 input attachment 索引
        //   用于替换 subpass 边界天然完成的「color attachment → input attachment」切换
        const uint32_t inputIndices[2] = { 0, 1 };   // G-Buffer 0→input 0, 1→input 1
        vk::RenderingInputAttachmentIndexInfo inputInfo{};
        inputInfo.setColorAttachmentCount(2)
                 .setPColorAttachmentInputIndices(inputIndices);

        // ★ color 附件定位重映射：让 shader 认为「最终颜色」位于 location 0
        const uint32_t colorLocations[1] = { 0 };
        vk::RenderingAttachmentLocationInfo locationInfo{};
        locationInfo.setColorAttachmentCount(1)
                    .setPColorAttachmentLocations(colorLocations);

        // 最终颜色附件（本 pass 的唯一 color attachment）
        vk::RenderingAttachmentInfo finalColor{};
        finalColor.setImageView(m_finalColorView)
                  .setImageLayout(vk::ImageLayout::eColorAttachmentOptimal)
                  .setLoadOp(vk::AttachmentLoadOp::eClear)
                  .setStoreOp(vk::AttachmentStoreOp::eStore);

        vk::RenderingInfo ri{};
        ri.setRenderArea(m_fullArea)
          .setLayerCount(1)
          .setColorAttachments(finalColor)
          .setPNext(&inputInfo);          // ★ 挂载 input 索引映射

        // 这两个重映射也可以在录制期用命令设置（适用于需要动态切换的场景）
        cmd.setRenderingInputAttachmentIndices(inputInfo);
        cmd.setRenderingAttachmentLocations(locationInfo);

        cmd.beginRendering(ri);
        cmd.bindPipeline(vk::PipelineBindPoint::eGraphics, m_lightingPipeline);
        cmd.draw(3, 1, 0, 0);             // 全屏三角形：无害，访问是 framebuffer-local 的
        cmd.endRendering();
    }
}
```

**Shader 侧完全不变**：

```glsl
// 与 subpass 版本完全一致，无需任何修改
layout(input_attachment_index = 0, set = 0, binding = 0) uniform subpassInput gbuf0;
layout(input_attachment_index = 1, set = 0, binding = 1) uniform subpassInput gbuf1;

void main() {
    vec4 rt0 = subpassLoad(gbuf0);
    vec4 rt1 = subpassLoad(gbuf1);
    // ...
}
```

⚠️ **注意 `vkCmdSetRenderingInputAttachmentIndicesKHR` 的额外能力**：它允许**把 input attachment 索引直接重映射到绑定的描述符**，而不必是 render pass instance 的 color/depth/stencil 附件。这比 subpass 更灵活——subpass 的 input attachment 必须来自同一个 render pass。

---

## 7. 是不是「更彻底地适配移动端」

### 7.1 结论

**方向对，但要打折扣。** 准确表述：

```
┌────────────────────────────────────────────────────────────────────────────────┐
│  动态渲染 + local read 在移动端带来了什么                                        │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  ✅ 保留的能力                                                                  │
│     · framebuffer-local 读（input attachment 的片上语义）                       │
│     · 可以用 BY_REGION barrier 表达「同一像素内」的依赖                          │
│     · shader 接口不变 → 移动端已经调好的 shader 逻辑不用重写                      │
│                                                                                │
│  ✅ 新增的收益                                                                  │
│     · 不必预创建 render pass / framebuffer 对象 → 大幅降低 API 与内存管理负担     │
│     · 附件集合可随时变化 → 天然适配帧图系统的动态配置                             │
│     · 索引重映射比 subpass 更灵活（input 可直接指向描述符）                       │
│     · 可以自由地在 render pass instance 内插入 BY_REGION barrier                │
│                                                                                │
│  ⚠️ 没有变好的地方（重要！）                                                     │
│     · 片上闭环仍然完全依赖 BY_REGION → 漏掉照样失效，与 subpass 时代一样          │
│     · 不会自动选择「更好的渲染模式」→ 不解决 Adreno Direct Mode 问题             │
│     · depth/stencil/MSAA 的 local read 是可选特性 → 可能不提供                   │
│     · 未复刻的 subpass 功能需拆 MRT pass → 反而可能增加 Resolve                  │
│     · VK_QCOM_render_pass_shader_resolve 无等价物 → 用它就回不去                │
│                                                                                │
│  ❌ 完全不涉及的事                                                              │
│     · 不改变 Adreno 的 FlexRender 模式选择逻辑                                   │
│     · 不改变 tile 尺寸、不扩大 GMEM                                             │
│     · 不减少 Resolve 成本                                                       │
│     · 不解决分箱阶段的图元/顶点压力                                              │
└────────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 对移动端最有价值的三个实际改进

| 改进                                                   | 为什么移动端特别受益                                                                                                         |
| ------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------- |
| **不再需要预创建对象**                           | 移动端内存紧张。一个后处理链可能有几十种附件组合 → 几十个 render pass 对象 + framebuffer 对象，每个都是有开销的驱动内部结构 |
| **可以在 render pass instance 内自由插 barrier** | 移动端管线常需要在同一渲染过程中切换多套状态（例如先做不透明再做 alpha test），旧 subpass 模型要求预先精确规划子通道划分     |
| **附件集合动态化**                               | 移动端机型差异大（有的支持 UBWC、有的 tile memory 更小），需要按设备能力切换附件配置。旧模型要为每种组合建一套 render pass   |

---

## 8. 三个相关扩展的准确定位

⚠️ **这是资料错得最厉害的部分，需要单独厘清。**

### 8.1 读取粒度是理解三者的钥匙

```
┌────────────────────────────────────────────────────────────────────────────────┐
│  三种「片上读取」的粒度完全不同                                                  │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  ① subpass input attachment / dynamic rendering local read                     │
│     ┌──────────────────────┐                                                   │
│     │ ░░▓▓▒▒░░ ░░▓▓▒▒░░   │  读取范围：**仅当前像素位置**                     │
│     │ ░░▓▓▒▒░░ ░░▓▓▒▒░░   │  读取对象：前一个写阶段（barrier 之前）写入的附件   │
│     │ ░░▓▓▒▒░░ [▓]▓▒▒░░   │  用途：几何 → 光照、后处理链                       │
│     │ ░░▓▓▒▒░░ ░░▓▓▒▒░░   │                                                   │
│     └──────────────────────┘                                                   │
│                                                                                │
│  ② VK_EXT_shader_tile_image                                                    │
│     ┌──────────────────────┐                                                   │
│     │ ░░▓▓▒▒░░ ░░▓▓▒▒░░   │  读取范围：**仅当前像素位置**                     │
│     │ ░░▓▓▒▒░░ ░░▓▓▒▒░░   │  读取对象：color / depth / stencil 的当前值        │
│     │ ░░▓▓▒▒░░ [▓]▓▒▒░░   │            （rasterization order 保证）           │
│     │ ░░▓▓▒▒░░ ░░▓▓▒▒░░   │  用途：programmable blending、                               │
│     └──────────────────────┘        framebuffer fetch 的 Vulkan 等价物           │
│                                                                                │
│  ③ VK_QCOM_tile_shading                                                        │
│     ┌──────────────────────┐                                                   │
│     │ ▓▓▓▓▓▓▓▓ ▓▓▓▓▓▓▓▓   │  读取范围：**整个 tile（+ apron）**               │
│     │ ▓▓▓▓▓▓▓▓ ▓▓▓▓▓▓▓▓   │  读取对象：per-tile view 的 color/depth/input     │
│     │ ▓▓▓▓▓▓▓▓ ▓▓▓▓▓▓▓▓   │  attachment（可任意坐标读写/采样）                 │
│     │ ▓▓▓▓▓▓▓▓ ▓▓▓▓▓▓▓▓   │  执行单元：**fragment 和 compute 都可以**          │
│     └──────────────────────┘  用途：GPU-Driven 的 per-tile 处理               │
│                                                                                │
│  → ① ② 是「像素局部」，③ 是「tile 局部」。粒度差一个量级 → 用途完全不同。        │
└────────────────────────────────────────────────────────────────────────────────┘
```

### 8.2 各扩展的准确定位

| 扩展                                    | 准确定位（官方口径）                                                                                                                                                                                      | 常见误解                                                      |
| --------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| `VK_KHR_dynamic_rendering_local_read` | 让 dynamic rendering 具备 subpass 的 pixel local read 能力，**替代 subpass 依赖**。Vulkan 1.4 核心                                                                                                  | 误以为它是「framebuffer fetch」的同义词                       |
| `VK_EXT_shader_tile_image`            | **OpenGL `GL_EXT_shader_framebuffer_fetch` 的 Vulkan 等价物**。引入 "tile image" 概念，fragment shader 可读**当前 fragment 位置**的 color/depth/stencil。**只用于 dynamic rendering** | 误以为它能做 tile 级操作（实际读的仍是当前像素位置）          |
| `VK_QCOM_tile_shading`                | 提供**per-tile view** 的附件访问（`TileAttachmentQCOM` 存储类），fragment 与 compute 均可用。**其功能是 `VK_EXT_shader_tile_image` 的超集**（除 descriptor-less access 外）               | ❌**误以为它是「禁用 FlexRender / 强制 TBDR」的开关**   |
| `VK_QCOM_tile_memory_heap`            | 允许把 image/buffer**显式分配**到 tile memory heap 并跨 pass 常驻                                                                                                                                   | 误以为可有可无——官方要求它与 tile_shading**配套使用** |
| `VK_QCOM_render_pass_store_ops`       | 提供`VK_ATTACHMENT_STORE_OP_NONE_QCOM`（真正零写回语义）                                                                                                                                                | 误以为等价于`DONT_CARE`（后者仍允许实现写回）               |

**关于 `VK_QCOM_tile_shading` 与 `VK_EXT_shader_tile_image` 的关系**，官方提案里有一条直接的问答：

> "RESOLVED: The functionality of this extension is a **superset** of `VK_EXT_shader_tile_image`. `VK_EXT_shader_tile_image` is limited to bringing the functionality of `GL_EXT_shader_framebuffer_fetch` to Vulkan dynamic render passes. The associated `SPV_EXT_shader_tile_image` and `GL_EXT_shader_tile_image` extensions provide **descriptor-less read-only access to only the current fragment location for only color/depth/stencil attachments**. This extension is a superset of the functionality in `VK_EXT_shader_tile_image` **with the exception of descriptor-less access**."

### 8.3 `VK_QCOM_tile_shading` 到底是什么（官方约束）

| 维度                           | 事实                                                                                                                                                                                                                                                                                                                                                                                                                   |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 提供的能力                     | shader 以**per-tile view** 访问 color / depth / input attachment                                                                                                                                                                                                                                                                                                                                                 |
| 支持的 stage                   | **compute**（全部能力）+ **fragment**（需 `tileShadingFragmentStage` 特性）                                                                                                                                                                                                                                                                                                                              |
| 读取方式                       | sampled tile attachment（`OpImageFetch` / `OpImageSample*` 等）、storage tile attachment（`OpImageRead` / `OpImageWrite`）、input tile attachment（`OpImageRead`）                                                                                                                                                                                                                                           |
| ⚠️ 写入限制                  | `OpImageWrite` **只允许在 compute stage**；fragment shader **不得**把 color attachment 当 storage image 做 load/store；compute 与 fragment 都**不得**向 depth/stencil、resolve、input attachment 写入                                                                                                                                                                                              |
| ⚠️ 越界                      | 访问必须在**tile 边界（+ apron）内**，越界是 **UB**。sampler 的 clamp/wrap 作用在 **VkImage 边缘而非 tile 边缘** → 需要 shader 自己 clamp                                                                                                                                                                                                                                                           |
| ⚠️ descriptor 要求           | tile attachment 必须由与`VkRenderingAttachmentInfo` / `VkFramebuffer` 中 `VkImageView` 等价的 descriptor 支撑（除 `aspectMask` 外）                                                                                                                                                                                                                                                                            |
| ⚠️ 配套要求                  | 官方明确要求**同时启用 `VK_QCOM_tile_memory_heap`**；「Using just `VK_QCOM_tile_shading` alone is **never recommended**, because then the driver tends to use far less GMEM than it otherwise could」                                                                                                                                                                                                  |
| ⚠️ 与 Direct Mode 的真实关系 | 官方原文：「In the unlikely event that you find the driver employing '**Direct**' Render Mode on a surface that uses a per-tile block, **performance on the per-tile block commands is likely very poor. Every effort should be made to discourage the driver from using 'Direct' Render Mode during per-tile blocks.**」→ **Direct Mode 是 per-tile 命令的敌人，而 tile_shading 不是阻止它的开关** |

### 8.4 关于「强制 TBDR 模式」的两重错误

```
┌────────────────────────────────────────────────────────────────────────────────┐
│  错误一：概念错误 —— Adreno 不是 TBDR                                          │
│    · TBDR = Tiling Accelerator + ISP（pixel-perfect HSR）+ TSP，是 Imagination │
│      的架构名                                                                  │
│    · Adreno 是 FlexRender：Direct / Binned / Binned Direct 三档动态切换       │
│    · Adreno 的隐面剔除是 LRZ（低精度深度）+ Early-Z，不是 HSR                  │
│    · 用 "TBDR" 描述 Adreno 是术语混淆                                          │
│                                                                                │
│  错误二：功能错误 —— tile_shading 不是渲染模式开关                              │
│    · 它是一个「让 shader 以 tile 为单位访问附件」的能力扩展                      │
│    · 官方文档中没有任何「禁用 FlexRender」的表述                                │
│    · 官方文档反而在教你怎么「劝阻驱动进入 Direct Mode」                          │
│    · 如果真有一个能强制 TBDR 的开关，Direct Mode 警告早就不是问题了              │
└────────────────────────────────────────────────────────────────────────────────┘
```

---

## 9. 迁移决策与路径

### 9.1 该不该迁移

```
                        开始
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 是否依赖任何「在 dynamic rendering 下无等价物」的  │
        │ 厂商扩展？                                     │
        │  · VK_QCOM_render_pass_shader_resolve         │
        │  · 其他需要 render pass 对象的功能              │
        └──────┬────────────────────────┬──────────────┘
               │ 是                      │ 否 / 不确定
               ▼                         ▼
        ┌─────────────────┐    ┌──────────────────────────────────┐
        │ 保留 render pass │    │ 目标平台是否支持                   │
        │ 路径，不要迁移    │    │ VK_KHR_dynamic_rendering_local_read│
        │（或双路径并存）   │    └──────┬───────────────┬──────────┘
        └─────────────────┘           │ 是            │ 否（或需兼容旧设备）
                                      ▼               ▼
                        ┌──────────────────────┐  ┌────────────────────┐
                        │ 迁移到 dynamic        │  │ 阶段 1：先迁移到    │
                        │ rendering + local read│  │ dynamic rendering  │
                        │                      │  │（不含 local read）  │
                        │                       │  │ 阶段 2：设备支持时  │
                        │                       │  │ 启用 local read 路径│
                        └──────────────────────┘  └────────────────────┘
```

### 9.2 迁移检查清单

```
□ 阶段 0：能力探测
    □ 查询 VkPhysicalDeviceDynamicRenderingLocalReadFeaturesKHR.dynamicRenderingLocalRead
    □ 查询 shaderTileImage* / tileShading* 等相关特性
    □ 确认是否依赖无等价物的厂商扩展（已知：VK_QCOM_render_pass_shader_resolve）
    □ 确认是否需要 depth/stencil/MSAA 的 local read（⚠️ 可选特性，可能不支持）

□ 阶段 1：结构替换
    □ vkCmdBeginRenderPass → vkCmdBeginRendering
    □ VkRenderPassBeginInfo → VkRenderingInfo
    □ VkFramebuffer 引用 → VkRenderingAttachmentInfo（栈上）
    □ 管线创建：VkRenderPass → VkPipelineRenderingCreateInfo
    □ 删除 VkRenderPass / VkFramebuffer 的创建与销毁代码

□ 阶段 2：subpass → barrier
    □ vkCmdNextSubpass → vkCmdPipelineBarrier2（★ 带 eByRegion）
    □ 附件布局 → VK_IMAGE_LAYOUT_RENDERING_LOCAL_READ_KHR
    □ barrier 的 oldLayout == newLayout（禁止 layout transition）
    □ barrier 的 stage 全部是 framebuffer-space
    □ 需要索引重映射时使用 vkCmdSetRenderingInputAttachmentIndicesKHR /
       vkCmdSetRenderingAttachmentLocationsKHR

□ 阶段 3：验证
    □ 带宽实测对比迁移前后（重点是 G-Buffer 是否仍留在片上）
    □ 若仍可用 render pass 路径，用 VK_EXT_subpass_merge_feedback 做交叉验证
    □ 确认没有引入新的 layout transition
    □ 确认 depth/stencil 的 storeOp 仍为 DONT_CARE

□ 阶段 4：Shader
    □ subpassInput / subpassLoad 无需修改（这是迁移的最大便利）
    □ 若改用 tile image / tile attachment，需要重写为 OpColorAttachmentReadEXT /
       TileAttachmentQCOM 语义
```

### 9.3 迁移的收益预期管理

| 预期收益                        | 是否成立             | 说明                                                                                                     |
| ------------------------------- | -------------------- | -------------------------------------------------------------------------------------------------------- |
| 代码更简洁、对象更少            | ✅ 成立              | 这是主要收益                                                                                             |
| 帧图系统集成更容易              | ✅ 成立              | 附件集合可动态变化                                                                                       |
| 内存占用下降                    | ✅ 成立              | 少了几十到几百个 render pass / framebuffer 对象                                                          |
| **性能提升**              | ⚠️**不一定** | 片上行为不变（同样依赖 BY_REGION）。性能若有变化，通常来自对象管理开销或驱动的优化路径差异，而非渲染语义 |
| **解决 Direct Mode 警告** | ❌ 不成立            | Direct Mode 是 surface 级驱动启发式，与 API 选择无关                                                     |
| **减少 Resolve 成本**     | ❌ 不成立            | Resolve 由「渲染到贴图」这个动作决定，与 API 无关                                                        |

⚠️ **这条很重要**：迁移到 dynamic rendering 的正当理由是**工程可维护性**，不是性能。如果你的 Direct Mode 警告没有解决，那是另一个问题（见第 5 篇的诊断树）。

---

## 10. 勘误表

| #  | 原文表述                                                                            | 正确表述                                                                                                                                                 |
| -- | ----------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1  | 「Vulkan 1.3 将`VkRenderPass` 标记为不推荐使用」                                  | **Vulkan 1.4** 弃用。1.3 只是把 dynamic rendering 提升为核心                                                                                       |
| 2  | 「1.3 引入动态渲染，通过一系列扩展给出 subpass 的答案」                             | 1.3 的 dynamic rendering**缺** subpass 等价物；1.4 才由 `VK_KHR_dynamic_rendering_local_read` 补上                                               |
| 3  | 「动态渲染完美复现了 Subpass 的核心优势」                                           | 复现了**大部分**（most）。未复刻的部分需要拆成多个 render pass instance；`VK_QCOM_render_pass_shader_resolve` 无等价物                           |
| 4  | 「`VK_QCOM_tile_shading` 启用后会禁用 FlexRender，强制 GPU 完全运行在 TBDR 模式」 | 无任何官方依据，且与真实用途相反。它提供 per-tile 附件访问能力；官方要求「尽一切努力劝阻 Direct Mode」                                                   |
| 5  | （隐含）Adreno 是 TBDR                                                              | Adreno 是**FlexRender**（三档动态切换），不是 TBDR。TBDR 是 Imagination 的术语                                                                     |
| 6  | 「`VK_EXT_shader_tile_image` 为 Tile 内的精细控制提供了更底层的灵活性」           | 它的读粒度**仍是当前像素位置**，不是 tile 级。定位是 `GL_EXT_shader_framebuffer_fetch` 的 Vulkan 等价物                                          |
| 7  | 「将渲染目标的指定从管线创建阶段提前到命令录制阶段」                                | 从「render pass 对象（预先创建的独立对象）」转移到「`VkRenderingInfo`（命令录制期栈上结构）」。管线仍需通过 `VkPipelineRenderingCreateInfo` 声明格式 |
| 8  | 「Vulkan 需要在同一套 API 下同时服务桌面与移动，导致设计僵化」                      | 这是次要因素。Khronos 给出的主要理由是**setup 繁琐、难以融入动态变化的管线、对引擎不友好**                                                         |
| 9  | 「必须启用 local read 扩展，否则你将失去 Tile 架构带来的所有带宽优势」              | 「必须」应限定为**需要使用 input attachment 的场景**。纯 forward 管线单 pass 渲染到屏幕，动态渲染本身足够                                          |
| 10 | 「Local Read 的读粒度是片上 Tile 内存」                                             | 读粒度是**当前像素位置**。tile 级读取需要 `VK_QCOM_tile_shading` 的 tile attachment                                                              |
| 11 | 「你现在遇到的 Direct Mode 警告，说明正确配置这些渲染路径的重要性」                 | Direct Mode 的**官方首要触发条件**是 VS 中纹理采样、tessellation/GS、顶点/draw 数少。渲染路径配置只是部分相关                                      |

---

## 11. 小结

| 要点                                                                  | 说明                                                                                                                                               |
| --------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| **弃用发生在 Vulkan 1.4，不是 1.3**                             | 1.3 只是把`VK_KHR_dynamic_rendering` 提升为核心；`VkRenderPass` 当时仍完全推荐                                                                 |
| **1.3 的 dynamic rendering 恰好缺了移动端最需要的东西**         | Khronos 官方承认：「the original extension didn't address input attachments or subpasses —**Critical for performance on tile-based GPUs**」 |
| **补上这块的是 1.4 的 `VK_KHR_dynamic_rendering_local_read`** | 它让 dynamic rendering 能表达**大部分** subpass 功能。着色器接口不变（仍是 `subpassLoad`）                                                 |
| **「完整替代」不成立，规范里留了明确缺口**                      | 未复刻的功能需拆多个 render pass instance；`VK_QCOM_render_pass_shader_resolve` **没有**等价物，用它就仍需已弃用功能                       |
| **deprecated ≠ 失效**                                          | 在下一个 major core 版本之前必须继续可用。风险在于「新功能可能不定义与已弃用功能的交互」                                                           |
| **`VK_QCOM_tile_shading` 不是渲染模式开关**                   | 它提供 per-tile 附件访问能力（fragment + compute）。官方要求它与`VK_QCOM_tile_memory_heap` 配套，且要「劝阻 Direct Mode」而非「禁用 FlexRender」 |
| **三种片上读取的粒度不同**                                      | local read 与 shader_tile_image 都是**当前像素位置**；只有 `tile_shading` 是**整个 tile**。粒度差一个量级，用途完全不同              |
| **迁移的正当理由是工程可维护性，不是性能**                      | 片上闭环仍完全依赖`VK_DEPENDENCY_BY_REGION_BIT`，与 subpass 时代一致。它不解决 Direct Mode、不减少 Resolve                                       |
| **迁移前必须先查「无等价物」的依赖**                            | 尤其是`VK_QCOM_render_pass_shader_resolve`。双路径并存常常是更实际的选择                                                                         |
| **迁移成本主要在 CPU 侧代码，不在 Shader**                      | 这是这次 API 演进最友好的地方——移动端已调好的 shader 逻辑可以原样保留                                                                            |

---

*上一篇：[05-性能诊断与优化清单](05-性能诊断与优化清单.md) | 下一篇：[07-与主机端的优化差异对照](07-与主机端的优化差异对照.md)*
