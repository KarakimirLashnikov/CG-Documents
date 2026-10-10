# 阶段一（M1–M3）— UE渲染管线与RHI定制

> 本文档是「资深引擎开发成长路线」系列文档的第 **1** 篇。覆盖能力域 **A（渲染管线架构）+ C（RHI 与平台层）**：从 UE 5.8 延迟渲染主线出发，经 RDG 实操下钻到 Vulkan RHI 后端，最终具备「在 UE 上落地自定义渲染特性」的完整链路能力。
>
> ⚠️ 文中源码路径均相对 `Engine/Source`，以 UE 5.8.2 本地源码树为准；行号会随 hotfix 漂移，以符号名为准。

---

## 1. 阶段目标与验收

| 目标 | 验收标准 |
|------|---------|
| 建立 UE 渲染主线全局认知 | 能默画 `FDeferredShadingSceneRenderer::Render` 的 Pass 序列与数据依赖图 |
| RDG 实操能力 | 独立实现一个自定义 RDG Pass 并接入主线，RenderDoc 可见、资源依赖正确 |
| RHI 链路理解 | 讲清一次 Draw 从 RDG 录制到 `vkCmdDraw*` 的完整调用链 |
| Vulkan 后端定制入门 | 能在 VulkanRHI 层加 instrumentation / 小特性并验证 |

---

## 2. M1：渲染主线地图建立（学习为主）

### 2.1 周计划

| 周 | 主题 | 具体动作 | 产出 |
|----|------|---------|------|
| W1 | 三层时间尺度与核心类型 | 精读 `FScene` / `FPrimitiveSceneInfo` / `FSceneView` / `FViewInfo` 的职责与归属 | 类型职责卡片（每类型 5 行内） |
| W2 | 可见性与绘制命令 | 剔除 → Relevance → `FMeshDrawCommand` 生成；静态缓存 vs 动态收集 | 数据流笔记 + 断点验证记录 |
| W3 | 延迟主线 Pass 序列 | PrePass/Depth → Nanite → BasePass(GBuffer) → Shadow → Lumen → Lights → Translucency → Post | **自绘 Pass 依赖全图**（里程碑） |
| W4 | GT/RT/RHI 三线程模型 | `ENQUEUE_RENDER_COMMAND`、命令队列、帧边界与同步 | 线程模型图 + 常见竞态坑清单 |

### 2.2 阅读纪律（防黑洞）

```
读任何一段渲染代码前，先回答三个问题：
  ① 这段代码在哪个线程、哪个时间尺度上执行？
  ② 它读的数据是谁在什么时候填的？
  ③ 它写的数据被谁、在哪个 Pass 消费？
答不出 ② 就回溯数据源头，答不出 ③ 就先跳过标记书签。
```

**教材**：`UE渲染源码导读文档/` 全系列（00 总览 → 01/02 核心概念 → 05 延迟主线 → 06 分支识别）。主线函数只有一条：`FDeferredShadingSceneRenderer::Render`（DeferredShadingRenderer.cpp），其余全是旁枝。

### 2.3 配套实验（W3）

- 用 RenderDoc 截一帧 UE 编辑器 standalone 运行帧，对照源码逐个 Pass 核对：Event 名称 ↔ RDG Pass 名 ↔ 源码函数；
- ⚠️ 坑：编辑器帧含 Slate/UI Pass，噪音大——用 `-game` 模式或空关卡截帧更干净。

---

## 3. M2：RDG 实操——落地自定义 Pass（实践为主）

### 3.1 最小自定义 Pass 骨架

在插件中实现一个全屏自定义 Pass（示例：自定义色调分离/边缘检测），接入点选 `PostProcessPass` 之前：

```cpp
// ① 声明 RDG 参数结构（Shader 参数与 Pass 资源合一）
BEGIN_SHADER_PARAMETER_STRUCT(FMyCustomPassParameters, )
    SHADER_PARAMETER_RDG_TEXTURE_SRV(Texture2D, SceneColor)
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2D, Output)
    SHADER_PARAMETER(FVector4f, TintParams)
END_SHADER_PARAMETER_STRUCT()

// ② 在渲染扩展点注入（5.x 推荐方式，避免改引擎源码）
void FMySceneViewExtension::PrePostProcessPass_RenderThread(
    FRDGBuilder& GraphBuilder, const FSceneView& View,
    const FPostProcessingInputs& Inputs)
{
    FRDGTextureRef SceneColor = Inputs.SceneTextures->GetSceneColorTexture();

    auto* PassParams = GraphBuilder.AllocParameters<FMyCustomPassParameters>();
    PassParams->SceneColor = GraphBuilder.CreateSRV(
        FRDGTextureSRVDesc::Create(SceneColor));
    PassParams->Output = GraphBuilder.CreateUAV(MyOutput);
    PassParams->TintParams = FVector4f(1.0f, 0.9f, 0.8f, 16.0f);

    TShaderMapRef<FMyCustomCS> CS(View.ShaderMap);
    FComputeShaderUtils::AddPass(GraphBuilder,
        RDG_EVENT_NAME("MyCustomPass"),       // ⚠️ 必须命名，RenderDoc/Insights 可见性全靠它
        CS, PassParams,
        FComputeShaderUtils::GetGroupCount(
            SceneColor->Desc.Extent, FIntPoint(8, 8)));
}
```

⚠️ 要点与坑：

| 坑 | 说明与对策 |
|----|-----------|
| 资源未声明依赖 | 所有读写必须经 `CreateSRV/CreateUAV/AddPass` 声明，否则 RDG 屏障错误——表现为偶发花屏，极难排查 |
| 扩展点选错 | `ISceneViewExtension` 各回调顺序文档稀缺，以 `SceneViewExtension.cpp` 源码顺序为准 |
| `RDG_EVENT_NAME` 缺失 | Pass 在 RenderDoc 里匿名，后期性能分析自找麻烦 |
| 引擎源码 vs 插件 | 能走 `ISceneViewExtension` / `FRendererModule` 扩展点就不改引擎——升级 5.9/UE6 时省一条命 |

### 3.2 进阶实验（W7–W8）

- 实验二：实现一个 **compute-based 自定义剔除 Pass**（HiZ 遮挡查询简化版），输出间接绘制参数，验证 RDG 的 buffer 依赖与 UAV barrier；
- 实验三：把同一效果在 **RenderToy（自研 Vulkan FrameGraph）** 里实现一遍，对照两框架的资源声明/屏障推导差异，写一篇对照笔记。

### 3.3 周计划

| 周 | 动作 | 产出 |
|----|------|------|
| W5 | 搭插件骨架 + `ISceneViewExtension` 接入 + 全屏 CS Pass 跑通 | 可运行插件 v0.1 |
| W6 | RenderDoc 截帧验证资源依赖；补 `RDG_EVENT_NAME`；写落地文档 | 里程碑 M2 验收 |
| W7 | 自定义剔除 Pass（间接绘制） | 插件 v0.2 |
| W8 | RenderToy 对照实现 + 对照笔记 | 框架对照文档 |

---

## 4. M3：下钻 RHI——Vulkan 后端链路（深读 + 微定制）

### 4.1 一次 Draw 的完整调用链（必须能默写）

```
渲染线程 (RT)                          RHI线程 (RHIT)
─────────────                          ─────────────
FRDGBuilder::Execute()
  └► 逐 Pass: Pass->Execute(FRHICommandList&)
       │  (FRHIComputeCommandList)
       ▼
RHICmdList.SetGraphicsPipelineState()
RHICmdList.SetShaderParameter()      ──►  FVulkanCmdBuffer
RHICmdList.DrawIndexedPrimitive()         ├ 状态缓存 & 去重 (FVulkanStateCache)
       │  记录 RHI 命令                    ├ DescriptorSet 分配/绑定 (FVulkanDescriptorSets)
       ▼                                  ├ Layout 迁移 (FVulkanLayoutManager)
IRHICommandContext 实现                     ▼
       │                              vkCmdBindPipeline / vkCmdBindDescriptorSets
       ▼                              vkCmdDrawIndexed()
RHIThread 提交 ──► FVulkanQueue::Submit ──► vkQueueSubmit
```

关键源码位置（`Runtime/VulkanRHI/Private/`）：

| 主题 | 文件 |
|------|------|
| 命令上下文 | `VulkanCommands.cpp`（`FVulkanCommandListContext`） |
| 状态缓存 | `VulkanStateCache.cpp`（去重逻辑，性能关键） |
| 描述符管理 | `VulkanDescriptorSets.cpp`（UE 的 bindless 策略） |
| 屏障与布局 | `VulkanBarriers.cpp` / `VulkanLayoutManager` |
| RDG→RHI 桥 | `Runtime/RenderCore/Private/RenderGraphPass.cpp` |

### 4.2 微定制实验（W11–W12，里程碑 M3）

任选一个落地：

1. **GPU 时间戳注入**：在 VulkanRHI 的 Pass 边界插入 `vkCmdWriteTimestamp`，把逐 Pass GPU 耗时导出成 CSV，与 Unreal Insights GPU Track 对拍；
2. **自定义 Vulkan 扩展接入**：选取一个驱动已支持的扩展（如 `VK_EXT_debug_utils` 深度标签），在 RHI 层封装并验证；
3. **Pipeline 缓存统计**：挂钩 `FVulkanStateCache`，统计 PSO 命中/未命中率，输出报告。

⚠️ 改 RHI 层 = 改引擎源码，务必：单独 git 分支 + 每个改动点写注释块（`// [CUSTOM] 原因/日期`），否则 5.9 合并时崩溃。

### 4.3 与 RenderToy 的对照清单

| 概念 | UE 5.8 | RenderToy | 对照收获 |
|------|--------|-----------|---------|
| 帧图 | RDG（`FRDGBuilder`） | FrameGraph | 资源声明 vs 自动推导 |
| 描述符 | `FVulkanDescriptorSets` + bindless | VMA + bindless | 池化策略差异 |
| 屏障 | `FRDGBarrier` + VulkanLayoutManager | 手动/DAG | 自动化程度 vs 可控性 |
| 命令录制 | RT 录制、RHIT 提交 | 单线程录制 | 多线程命令的同步成本 |

---

## 5. 本阶段资料清单

| 资料 | 类型 | 用途 |
|------|------|------|
| `UE渲染源码导读文档/`（本工作区） | 主教材 | M1 全程 |
| `实时渲染系统文档/15-UERDG屏障与同步机制源码解析` | 深读 | M2–M3 |
| UE 官方文档：Render Dependency Graph / Scene View Extension | 官方 | 扩展点语义 |
| GDC/SIGGRAPH：FrameGraph 原始分享（Frostbite, Yuriy O'Donnell 2017） | 论文/演讲 | 帧图思想源头（本工作区有 PDF） |
| Vulkan 规范 synchronization2 章节 | 规范 | 屏障语义核对 |

---

## 6. 小结

| 要点 | 说明 |
|------|------|
| M1 | 主线地图：三时间尺度 + Pass 序列 + 三线程模型 |
| M2 | RDG 实操：自定义 Pass 插件，扩展点注入，不改引擎 |
| M3 | RHI 下钻：Draw 全链路默写级掌握 + VulkanRHI 微定制 |
| 双线 | 每个概念与 RenderToy 对照，写对照笔记 |
| 最大坑 | 资源依赖漏声明 → 偶发花屏；改 RHI 不标 CUSTOM → 升级灾难 |
| 验收 | M3 末能向他人完整讲清 RDG→RHI→vkCmd 链路并演示定制成果 |

*上一篇：[00-总览与能力模型](00-总览与能力模型.md) | 下一篇：[02-阶段二-Shader体系与光照材质](02-阶段二-Shader体系与光照材质.md)*
