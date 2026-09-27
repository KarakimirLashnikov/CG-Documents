# UE渲染源码导读 — 核心概念：Primitive 与 MeshDrawCommand

> 本文档是「UE渲染源码导读」系列文档的第 **2** 篇。讲清渲染数据链：**游戏世界里的组件，如何一步步变成 GPU 可直接消费的绘制命令**。这条链是「数据从哪来」问题的最大头，基于 UE 5.8.2 源码实测。

---

## 1. 数据链全景

```
[游戏线程, 注册时]
UPrimitiveComponent                    ← 游戏世界对象 (PrimitiveComponent.h:307)
   │ CreateRenderState_Concurrent      (PrimitiveComponent.cpp:620)
   │   └─ FScene::AddPrimitive         (RendererScene.cpp:1341)
   ▼        创建渲染线程镜像
FPrimitiveSceneProxy                   ← 组件的渲染线程代理 (PrimitiveSceneProxy.h:291)
   │ 渲染线程: SetTransform / CreateRenderThreadResources
   ▼        登记进场景
FPrimitiveSceneInfo                    ← FScene 中的常驻记录 (PrimitiveSceneInfo.h:269)
   │ AddToScene → AddStaticMeshes      (PrimitiveSceneInfo.cpp:1822/1604)
   │   └─ Proxy->DrawStaticElements()  (PrimitiveSceneProxy.h:439)
   ▼        产出中间格式
FMeshBatch (FStaticMeshBatch)          ← 同材质同 VF 的绘制单元描述 (MeshBatch.h:364)
   │ CacheMeshDrawCommands             (PrimitiveSceneInfo.cpp:583)
   │   └─ FMeshPassProcessor::AddMeshBatch → BuildMeshDrawCommands
   ▼        按 Pass 翻译
FMeshDrawCommand                       ← 完整 draw call 描述 (MeshPassProcessor.h:1281)
   │ 每帧: 可见性命中 → 排序
   ▼
FVisibleMeshDrawCommand                ← 可见命令轻量包装 (MeshPassProcessor.h:1762)
   │ FParallelMeshDrawCommandPass::DispatchDraw
   ▼
RHI 提交 GPU
```

**记忆口诀**：组件有 **Proxy**（镜像），Proxy 在 Scene 里有 **Info**（档案），Info 产出 **Batch**（中间件），Batch 经 **Processor** 翻成 **Command**（终态），可见后包一层 **Visible** 排序下发。

---

## 2. 链上各环节导读卡

### 2.1 UPrimitiveComponent

| 栏目 | 内容 |
|------|------|
| **是什么** | 游戏线程的可渲染组件基类（PrimitiveComponent.h:307） |
| **何时需要关注** | 查「对象什么时候/为什么进入渲染世界」；自定义组件接入渲染 |
| **依赖哪些数据** | 组件自身变换、材质、`ShouldComponentAddToScene()` |
| **数据从哪来** | 游戏逻辑/Actor |
| **何时执行** | `CreateRenderState_Concurrent`（PrimitiveComponent.cpp:620）→ `World->Scene->AddPrimitive(this)`（:648）；变换更新走 `SendRenderTransform_Concurrent`（:655）；动态数据走 `SendRenderDynamicData_Concurrent`（ActorComponent.cpp:2274，dirty 时触发） |

### 2.2 FPrimitiveSceneProxy

| 栏目 | 内容 |
|------|------|
| **是什么** | 组件在渲染线程的镜像（PrimitiveSceneProxy.h:291），游戏线程不可见；子类化自组件的 `CreateSceneProxy()`（PrimitiveComponent.h:2395） |
| **何时需要关注** | 自定义渲染对象；查某个 mesh 的 FMeshBatch 是谁产的 |
| **依赖哪些数据** | 组件同步过来的变换/材质/自定义数据 |
| **数据从哪来** | 两个产出函数：`DrawStaticElements(FStaticPrimitiveDrawInterface*)`（:439，注册时调一次）与 `GetDynamicMeshElements(...)`（:504，每帧对可见动态图元调用） |
| **何时执行** | 静态路径：注册时一次；动态路径：每帧可见性任务中 |

### 2.3 FPrimitiveSceneInfo

| 栏目 | 内容 |
|------|------|
| **是什么** | FScene 中每个图元的常驻档案（PrimitiveSceneInfo.h:269）：`Proxy`（:277）、稳定 ID `PrimitiveComponentId`（:283）、`StaticMeshes`（:303）、`StaticMeshRelevances`（:300）、`StaticMeshCommandInfos`（:297）、八叉树 `OctreeId`（:310）、光照交互 `LightList`（:358） |
| **何时需要关注** | 追静态绘制命令缓存的生成/重建；查图元与灯光的交互 |
| **依赖哪些数据** | Proxy 产出的 FStaticMeshBatch |
| **数据从哪来** | `AddToScene`（PrimitiveSceneInfo.cpp:1822）→ `AddStaticMeshes`（:1604）→ `CacheMeshDrawCommands`（:583）；材质/几何变更走 `UpdateStaticMeshes`（:2130） |
| **何时执行** | 注册与重建时（渲染线程） |

⚠️ **勘误**：5.8 中不存在 UE4 旧名 `CachePathPrimitives`，对应概念就是上面 `StaticMeshes / StaticMeshRelevances / StaticMeshCommandInfos` 三件套。

### 2.4 FMeshBatch / FMeshBatchElement

| 栏目 | 内容 |
|------|------|
| **是什么** | CPU 侧中间格式：「同一材质 + 同一 VertexFactory 的一组绘制单元」（MeshBatch.h:364）；`FMeshBatchElement`（:231）是单次 draw 的索引/实例参数 + `PrimitiveUniformBuffer` |
| **何时需要关注** | 自定义 Proxy 输出几何；查多 LOD/多 section 怎么拆分 |
| **关键字段** | `Elements`（:366）、`VertexFactory`（:369）、`MaterialRenderProxy`（:372）、`LCI` 光照缓存（:375）、`MeshIdInPrimitive`（:387，半透明排序用）、`LODIndex`（:390）、`CastShadow`/`bUseForDepthPass`（:399/:401） |
| **数据从哪来** | `DrawStaticElements`（静态）或 `GetDynamicMeshElements`（动态） |
| **何时执行** | 静态：注册时；动态：每帧 |

### 2.5 FMeshDrawCommand（MDC）

| 栏目 | 内容 |
|------|------|
| **是什么** | 紧贴 RHI 之上的**完整 draw call 描述**：PSO ID + shader 绑定 + 顶点流 + 索引 + 绘制参数（MeshPassProcessor.h:1281） |
| **何时需要关注** | 绘制合批/排序/状态切换优化；查某次 draw 的资源绑定 |
| **关键字段** | `ShaderBindings`（:1288）、`VertexStreams`（:1289）、`IndexBuffer`（:1290）、`CachedPipelineId`（:1295）、`FirstIndex/NumPrimitives/NumInstances`（:1300-1302）、`PrimitiveIdStreamIndex`（:1319）、`StencilRef`（:1322） |
| **数据从哪来** | `FMeshPassProcessor::BuildMeshDrawCommands<>`（MeshPassProcessor.h:2307，实现在 .inl）把 FMeshBatch 按 EMeshPass 翻译而来 |
| **何时执行** | 静态：注册/重建时缓存；动态：每帧重建 |

⚠️ **勘误**：`FMeshDrawCommandSharedState`（UE4 类型）5.8 已移除，状态并入 MDC 本体与 `FMeshDrawShaderBindings`。

### 2.6 FMeshPassProcessor 与 EMeshPass

| 栏目 | 内容 |
|------|------|
| **是什么** | 把 FMeshBatch 翻译成**某个具体 Pass** 的 MDC 的虚基类（MeshPassProcessor.h:2258）；每 Pass 一个子类（FDepthPassMeshProcessor、FBasePassMeshProcessor…） |
| **何时需要关注** | 新增自定义 Mesh Pass；理解「同一个 mesh 为什么在每个 Pass 各有一条命令」 |
| **依赖哪些数据** | `MeshPassType`（:2262）、`Scene`（:2263）、动态路径专用的 `ViewIfDynamicMeshCommand`（:2265） |
| **何时执行** | 静态缓存时由 `FPassProcessorManager::CreateMeshPassProcessor` 创建；动态每帧由 `FDynamicPassMeshDrawListContext`（:1857）驱动 |

`EMeshPass::Type`（MeshPassProcessor.h:56）在 5.8 共 39 个 Pass，常用的：`DepthPass`、`BasePass`、`CSMShadowDepth`、`VSMShadowDepth`、`Velocity`、`Translucency*`（7 个变体）、`CustomDepth`、`NaniteMeshPass`、`LumenCard*`、`Distortion`。**同一个 FMeshBatch 会被每个相关 Pass 各翻译一次**——这就是「一套几何、多套命令」的来源。

### 2.7 FVisibleMeshDrawCommand 与排序

| 栏目 | 内容 |
|------|------|
| **是什么** | 已判定可见的命令的轻量包装（MeshPassProcessor.h:1762）：`MeshDrawCommand` 指针（:1796）+ `SortKey`（:1799）+ `StateBucketId`（:1809，动态实例化合批桶）+ `CullingPayload`（:1816） |
| **排序语义** | `FMeshDrawCommandSortKey`（:1536，64 位 union）：BasePass 按 `Masked→PixelShaderHash→VertexShaderHash`（材质聚类减状态切换）；半透明按 `Priority→Distance→MeshIdInPrimitive`（back-to-front） |
| **何时执行** | 可见性任务命中后收集，`FCompareFMeshDrawCommands`（:1826）排序，`FParallelMeshDrawCommandPass::DispatchDraw` 提交 |

---

## 3. 静态缓存 vs 动态收集：两条生成路径

这是理解 MDC「何时生成」的关键分叉：

| | 静态路径（Cached） | 动态路径（Dynamic） |
|--|--------------------|---------------------|
| 触发 | 注册时 / 材质几何变更时 | 每帧 |
| 入口 | `FPrimitiveSceneInfo::CacheMeshDrawCommands`（PrimitiveSceneInfo.cpp:583） | `FVisibilityTaskData::GatherDynamicMeshElements`（SceneVisibility.cpp:4608） |
| 来源 | `DrawStaticElements` → `StaticMeshes` | `GetDynamicMeshElements` → `FMeshElementCollector` |
| 存储 | `StaticMeshCommandInfos` + `Scene->CachedDrawLists` | `FDynamicMeshDrawCommandStorage`（每帧清空，TChunkedArray 保证指针稳定，MeshPassProcessor.h:1751） |
| 条件 | `SupportsCachingMeshDrawCommands`（VF 无逐帧/逐视图绑定变化） | 兜底路径，任何 proxy 都可走 |
| 典型对象 | StaticMesh、InstanceStaticMesh | SkeletalMesh、粒子、Cable、调试绘制 |

每帧汇合：静态命令经 `FStaticMeshBatchRelevance::CommandInfosMask` 命中收集，动态命令由 `FVisibilityTaskData::SetupMeshPasses`（SceneVisibility.cpp:4721）→ `FSceneRenderer::SetupMeshPass`（SceneRendering.cpp:5010）生成，最终统一落进 `FViewInfo::VisibleMeshDrawCommands`。

---

## 4. 小结

| 要点 | 说明 |
|------|------|
| 数据链 | Component → Proxy → SceneInfo → MeshBatch → MeshDrawCommand → VisibleMeshDrawCommand |
| 中间格式 | FMeshBatch 是 proxy 与 Pass 之间的解耦层，一套几何被每个 EMeshPass 各翻译一次 |
| 两条路径 | 静态注册时缓存一次；动态每帧重建，汇合于 SetupMeshPasses |
| 排序 | BasePass 按 shader/PSO 聚类；半透明按距离 back-to-front |
| 稳定 ID | `PrimitiveComponentId` 跨 re-register 稳定，是查图元的锚 |
| 勘误 | `FMeshDrawCommandSharedState`、`CachePathPrimitives`、`PrimitiveIdBufferIndex` 均已不存在（见 00 篇勘误表） |

---

*上一篇：[01-核心概念Scene与View](01-核心概念Scene与View.md) | 下一篇：[03-核心概念Substrate与MegaLights](03-核心概念Substrate与MegaLights.md)*
