# UE渲染源码导读 — 核心概念：Scene 与 View

> 本文档是「UE渲染源码导读」系列文档的第 **1** 篇。厘清渲染系统里最重要的一对划分：**Scene（场景，跨帧持久的世界镜像）** 与 **View（视角，每帧临时的相机+渲染态）**。绝大多数「这个数据从哪来」的困惑，最终都能归结到「它属于 Scene 侧还是 View 侧」。基于 UE 5.8.2 源码实测。

---

## 1. 一句话总览

| 概念 | 一句话 | 文件 |
|------|--------|------|
| `FSceneInterface` | Engine 模块暴露给游戏侧的「场景句柄」抽象接口 | Runtime/Engine/Public/SceneInterface.h:119 |
| `FScene` | 渲染器私有模块中真正的场景对象，**UWorld 在渲染线程侧的镜像** | Runtime/Renderer/Private/ScenePrivate.h:1624 |
| `FSceneViewFamily` | 一次渲染提交的「视角集合 + 全局渲染设置」容器 | Runtime/Engine/Public/SceneView.h:2320 |
| `FSceneView` | 一个相机视角的**纯数据**描述（矩阵、视口、FOV） | Runtime/Engine/Public/SceneView.h:1481 |
| `FViewInfo` | `FSceneView` + 渲染器私有的每帧渲染态（可见性结果、动态网格收集结果等） | Runtime/Renderer/Private/SceneRendering.h:1254 |
| `FSceneViewState` | **跨帧持久**的每视图状态（TAA 历史、曝光、遮挡查询） | Runtime/Renderer/Private/SceneViewState.h:59 |
| `FSceneRenderer` | 「一帧一次」的场景渲染作用域对象 | Runtime/Renderer/Private/SceneRendering.h:2218 |

---

## 2. Scene 侧：跨帧持久的世界镜像

### 2.1 所有权关系（先把这张图记住）

```
UWorld (游戏线程, 持久)
  │  World.h:1504 ── class FSceneInterface* Scene;
  ▼
FScene : public FSceneInterface        ← 跨帧持久，随 World 销毁
  ├─ UWorld* World            (反向指针, ScenePrivate.h:1629)
  ├─ Primitives / Lights / StaticMeshes / GPUScene / ...
  └─ TArray<FSceneViewState*> ViewStates   ← Scene 名下挂的持久视图状态

创建点: GetRendererModule().AllocateScene(...)
        调用处 World.cpp:2472 / 10165（World 初始化时）
```

### 2.2 FScene 导读卡

| 栏目 | 内容 |
|------|------|
| **是什么** | UWorld 的渲染线程侧镜像；场景中所有可渲染对象（图元、灯光、雾、天空、云、贴花）与派生加速结构（八叉树、GPUScene、距离场数据）的**统一容器** |
| **何时需要关注** | 追踪「某个对象为什么没被渲染/被谁渲染」；查数据注册与缓存重建时机；理解任何字段的来源（大概率源头在 Scene 上） |
| **依赖哪些数据** | 游戏线程侧 UPrimitiveComponent / ULightComponent 等的注册与更新命令 |
| **数据从哪来** | `FScene::AddPrimitive`（RendererScene.cpp:1341）等 Add/Remove/Update 系列函数，由组件在游戏线程触发、渲染线程执行 |
| **何时执行** | 生命周期与 UWorld 相同；每帧 `FScene::UpdateAllPrimitiveSceneInfos` 在 `OnRenderBegin`（SceneRendering.cpp:4198）内应用积压更新 |

### 2.3 FScene 主要数据成员（按用途分组）

| 分组 | 成员（类型，ScenePrivate.h 行号） | 用途 |
|------|----------------------------------|------|
| 图元 | `Primitives`（`TArray<FPrimitiveSceneInfo*>`, :1699）、`PrimitiveSceneProxies`（:1703）、`PrimitiveBounds`（:1705）、`PrimitiveOctree`（:2049） | 场景全部图元 + 空间加速结构 |
| 灯光 | `Lights`（`FSceneLightInfoArray`, :1784）、`SkyLight`/`SkyLightStack`（:1825/:1886）、`LocalShadowCastingLightOctree`（:2044） | 全部光源 |
| 静态网格 | `StaticMeshes`（`TSparseArray<FStaticMeshBatch*>`, :2000）、`CachedDrawLists[EMeshPass::Num]`（:1652） | 注册时缓存的绘制命令 |
| LOD | `SceneLODHierarchy`（`FLODSceneTree`, :2083） | HLOD 树 |
| 大气与云 | `ExponentialFogs`（:2003）、`SkyAtmosphere`（:2006）、`VolumetricCloud`（:2015） | 天空雾云 |
| 派生数据 | `GPUScene`（:1933）、`DistanceFieldSceneData`（:1979）、`IndirectLightingCache`（:1929） | GPU 场景/距离场/ILC |
| 持久资源 | `UniformBuffers`（`FPersistentUniformBuffers`, :1648）、`ParameterCollections`（:2080） | 跨帧 uniform buffer |
| 视图状态 | `ViewStates`（`TArray<FSceneViewState*>`, :1635） | 跨帧持久视图状态登记处 |

⚠️ **关键认知**：`FScene` 就是「渲染数据的统一容器」。04 篇会展开：几乎所有「数据来源隐秘」的问题，答案都是「注册时写进了 FScene 的某个成员」。

---

## 3. View 侧：三层结构

View 侧容易混淆，因为它由**三个类**叠成，职责严格分层：

```
FSceneViewFamily  ──聚合──►  FSceneView[]  ──继承──►  FViewInfo
(一次渲染提交)              (纯相机数据)              (+渲染器私有渲染态)
   SceneView.h:2320           SceneView.h:1481         SceneRendering.h:1254
```

### 3.1 FSceneViewFamily 导读卡

| 栏目 | 内容 |
|------|------|
| **是什么** | 一次渲染提交的上下文：多个 View（分屏/立体）、RenderTarget、ShowFlags、时间、帧号 |
| **何时需要关注** | 查全局开关（EngineShowFlags）、帧号/时间不对、分屏与 SceneCapture 行为差异 |
| **依赖哪些数据** | 游戏线程构造时填入：`Views`/`AllViews`（:2417/:2420）、`RenderTarget`（:2426）、`Scene`（:2430，**借用不拥有**）、`EngineShowFlags`（:2433）、`Time`（:2436）、`FrameNumber`（:2439） |
| **数据从哪来** | UGameViewportClient / SceneCapture 在游戏线程构造；`BeginRenderingViewFamily`（SceneRendering.cpp:5354）补填帧号与流送视点 |
| **何时执行** | 每帧创建一次，渲染结束即销毁 |

### 3.2 FSceneView 导读卡

| 栏目 | 内容 |
|------|------|
| **是什么** | 一个相机视角的**纯数据**：`ViewMatrices`（:1521）、`ViewLocation/ViewRotation`（:1524-1525）、视口矩形（:1513/:1516）、FOV（:1579）、`ViewUniformBuffer`（:1489）、回指 `Family`（:1484）与 `State`（:1486） |
| **何时需要关注** | 矩阵/投影相关问题；自定义渲染需要拿相机参数时 |
| **依赖哪些数据** | `FSceneViewInitOptions`（:1501，含注入 ViewState 的 `SceneViewStateInterface`，SceneView.h:183） |
| **数据从哪来** | 游戏线程由 PlayerCameraManager / SceneCapture 计算填入 |
| **何时执行** | 每帧随 Family 创建 |

### 3.3 FViewInfo 导读卡（阅读渲染器时出镜率最高的类型）

| 栏目 | 内容 |
|------|------|
| **是什么** | `class FViewInfo : public FSceneView`（SceneRendering.h:1254）。渲染器内部的视图对象 = 相机纯数据 + **每帧重建的渲染态** |
| **何时需要关注** | 几乎任何 Pass 源码里出现的 `View.XXX` 都是它；追可见性结果、动态网格、灯光列表时必读 |
| **依赖哪些数据** | 见下方成员表 |
| **数据从哪来** | **可见性任务填充**（05 篇）：`BeginInitViews` 系列函数 |
| **何时执行** | 每帧随 SceneRenderer 创建（`TArray<FViewInfo> Views` 值拷贝，SceneRendering.h:2187），帧末销毁 |

`FViewInfo` 在 FSceneView 之上追加的关键成员：

| 成员（SceneRendering.h 行号） | 写入者 | 用途 |
|------------------------------|--------|------|
| `ViewState`（:1269） | Render() 开头挂接（DeferredShadingRenderer.cpp:1828-1839） | 跨帧持久状态（TAA/曝光） |
| `CachedViewUniformShaderParameters`（:1272） | InitViews 阶段 | 每帧视图 uniform 参数本体 |
| `PrimitiveVisibilityMap`（:1275） | `FrustumCull`/遮挡剔除（SceneVisibility.cpp:860/3372） | 图元可见位图 |
| `PrimitiveViewRelevanceMap`（:1299） | `FRelevancePacket::ComputeRelevance`（SceneVisibility.cpp:1520） | 图元渲染相关性 |
| `StaticMeshVisibilityMap`（:1302） | 可见性任务 | 静态 mesh 可见位图 ⚠️旧名 `StaticMeshBatchVisibility` 已改 |
| `VisibleLightInfos`（:1360） | `ComputeLightVisibility`（SceneVisibility.cpp:5671-5680） | 本视图可见灯光 |
| `ViewMeshElements` / `DynamicMeshElements`（:1372/:1381） | `GatherDynamicMeshElements`（SceneVisibility.cpp:4608） | 动态收集的临时 FMeshBatch |

### 3.4 FSceneViewState：View 侧的「持久飞地」

View 本身每帧销毁，但 TAA 历史、曝光适应、遮挡查询结果必须跨帧保留——这就是 `FSceneViewState`（SceneViewState.h:59，接口 `FSceneViewStateInterface` 在 SceneManagement.h:118）。

- 登记在 `FScene::ViewStates`（ScenePrivate.h:1635）；
- 每帧 `Render()` 开头把 `View.ViewState` 与 Scene 双向挂接（DeferredShadingRenderer.cpp:1828-1839）；
- ⚠️ 旧教程的 `GetSceneViewStateInterface()` 已移除，直接访问 `FSceneView::State` 成员。

---

## 4. FSceneRenderer：把 Scene 和 View 拧在一起的人

```
FSceneRenderer (每帧: 创建 → Render() → 销毁)     SceneRendering.h:2218
  ├─ FScene* Scene              ← 借用，不拥有（Scene 属于 UWorld）
  ├─ FViewFamilyInfo ViewFamily ← 值拷贝
  └─ TArray<FViewInfo> Views    ← 拥有，值拷贝（FSceneView 升格为 FViewInfo）
        └─ FSceneViewState* ViewState ──► 指回 Scene 名下持久的 ViewState
```

源码注释（SceneRendering.h:2213-2216）明确了生命周期：游戏线程初始化 → 传给渲染线程 → `Render()` → 删除。具体创建在 `FSceneRenderProcessor::CreateSceneRenderers`（SceneRenderBuilder.cpp:475，按 `EShadingPath::Deferred` new 出 `FDeferredShadingSceneRenderer`，:516）。

---

## 5. Scene 与 View 的划分原则（本篇核心）

判断任何数据「该在 Scene 还是 View」，用三个问题：

| 问题 | 是 → Scene 侧 | 是 → View 侧 |
|------|--------------|-------------|
| 换一台相机还需要它吗？ | 需要（世界本身的数据） | 不需要（该相机的私有结果） |
| 下一帧还需要它吗？ | 需要（持久/缓存） | 不需要（每帧重算）→ 例外：需要跨帧的放 ViewState |
| 游戏线程会改它吗？ | 会（组件注册/更新） | 不会（游戏线程只填相机参数） |

**典型对照**：

| 数据 | 归属 | 理由 |
|------|------|------|
| 图元列表、灯光列表、静态 MDC 缓存 | Scene | 与相机无关、跨帧复用 |
| 视锥剔除结果、可见灯光列表 | View（FViewInfo） | 每台相机结果不同、每帧重算 |
| TAA 历史、曝光 | ViewState | 每台相机不同、但要跨帧 |
| 相机矩阵 | FSceneView | 纯数据，每帧由游戏线程给出 |
| GBuffer、SceneColor | 两者都不是 | RDG 纹理，帧图内临时（见 04 篇） |

---

## 6. 小结

| 要点 | 说明 |
|------|------|
| Scene = 世界镜像 | `FScene`（ScenePrivate.h:1624），UWorld 持有，跨帧持久，统一数据容器 |
| View 三层 | Family（提交上下文）→ FSceneView（相机纯数据）→ FViewInfo（+每帧渲染态） |
| 持久飞地 | 视图侧要跨帧的数据放 `FSceneViewState`，挂在 Scene 名下 |
| 拧合者 | `FSceneRenderer` 每帧创建，借用 Scene、拥有 Views |
| 划分三问 | 换相机还要吗 / 下帧还要吗 / 游戏线程改吗 |
| 勘误 | `GetSceneViewStateInterface()` 已移除；`StaticMeshBatchVisibility` 改名 `StaticMeshVisibilityMap` |

---

*上一篇：[00-系列总览与渲染知识地图](00-系列总览与渲染知识地图.md) | 下一篇：[02-核心概念Primitive与MeshDrawCommand](02-核心概念Primitive与MeshDrawCommand.md)*
