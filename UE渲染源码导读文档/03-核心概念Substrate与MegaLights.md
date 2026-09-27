# UE渲染源码导读 — 核心概念：Substrate 与 MegaLights

> 本文档是「UE渲染源码导读」系列文档的第 **3** 篇。Substrate 与 MegaLights 是 UE5 渲染器里两套「新架构」：一个重构**材质/GBuffer**，一个重构**直接光照**。它们都在主线里插了分叉点，第一遍读源码时很容易被打乱节奏。本篇目标：知道它们是什么、何时分叉、**以及第一遍阅读时能否跳过**。基于 UE 5.8.2 源码实测。

---

## 1. 定位速览

| | Substrate | MegaLights |
|--|-----------|-----------|
| 一句话 | 用 BSDF 组合（Slab 模型）取代固定 ShadingModel 的材质系统重构 | 用统一随机采样取代「每灯一个 draw/dispatch」的直接光照重构 |
| 改动层 | 材质编译 + GBuffer 布局 + 光照解算 | 灯光循环 + 阴影采样 |
| 源码目录 | `Runtime/Renderer/Private/Substrate/`（开关在 `Runtime/RenderCore/Private/RenderUtils.cpp`） | `Runtime/Renderer/Private/MegaLights/` |
| 5.8 默认状态 | **关**（`r.Substrate=0`，ECVF_ReadOnly；5.7+ 新建项目模板写入开启） | **关**（`r.MegaLights.EnableForProject=0`，注释明示 Experimental） |
| 主线阅读建议 | 第一遍可跳过，但要认得它的分叉点 | 第一遍可跳过，灯光排序里的区间划分要知道 |

---

## 2. Substrate 导读卡

| 栏目 | 内容 |
|------|------|
| **是什么** | 把材质从「枚举 ShadingModel（DefaultLit/ClearCoat/…）」重构为「BSDF Slab 树组合」：材质节点输出一个 Substrate 材质描述，引擎按组合出的 BSDF 类型生成 shader 排列与 GBuffer 编码 |
| **何时需要关注** | ① 项目开启了 Substrate（查 `r.Substrate` 或 DefaultEngine.ini）；② 读材质 shader 生成 / GBuffer 布局 / 复杂材质（车漆、分层）时 |
| **依赖哪些数据** | 材质图编译产物（Substrate 材质描述）、`r.Substrate.ProjectGBufferFormat`（默认 1=Adaptive/bitstream 编码；0=Blendable GBuffer） |
| **数据从哪来** | 材质侧 `MaterialGetSubstrateMaterialType_RenderThread`（BasePassRendering.cpp:1864）决定排列；帧数据由 `Substrate::InitialiseSubstrateFrameSceneData`（DeferredShadingRenderer.cpp:2400，每帧调、内部按开关短路） |
| **何时执行** | BasePass 阶段分叉（GBuffer 输出格式）+ BasePass 后分类 Pass |

### 2.1 开关判定链

```
Substrate::IsSubstrateEnabled()     RenderUtils.cpp:2221
   = CVarSubstrate(r.Substrate, 默认0, ReadOnly) > 0
        │
        ├─ 0 → 经典 ShadingModel 路径（老 GBuffer 布局）
        └─ 1 → Substrate 路径
              ├─ r.Substrate.ProjectGBufferFormat=1 → Adaptive/bitstream GBuffer
              └─ =0 → Blendable GBuffer（IsSubstrateBlendableGBufferEnabled, :2226）
```

### 2.2 主线分叉点（读代码时看到即认出）

| 位置 | 代码 | 作用 |
|------|------|------|
| 帧初始化 | `Substrate::InitialiseSubstrateFrameSceneData`（DeferredShadingRenderer.cpp:2400） | 每帧无条件调用，内部短路——**关闭时无成本** |
| BasePass RT 格式 | `Substrate::SetBasePassRenderTargetOutputFormat`（BasePassRendering.cpp:908） | 决定 GBuffer MRT 布局 |
| BasePass 绑定 | `AppendSubstrateMRTs`（:1013）、Substrate uniform 绑定（:757） | 输出目标与参数 |
| BasePass 后 | `AddSubstrateDBufferBasePass` / `AddSubstrateMaterialClassificationPass` / `AddSubstrateDBufferPass` / `AddSubstrateSampleMaterialPass`（DeferredShadingRenderer.cpp:3168-3190） | 材质分类：把 bitstream GBuffer 解析成逐材质解算 |
| 光照阶段 | `AddSubstrateOpaqueRoughRefractionPasses`（:3537） | 粗糙折射 |

⚠️ **阅读对策**：项目没开 Substrate 时，上述调用全部短路，读到 :3168 一段直接跳过即可；开了 Substrate 的项目，BasePass 的「写 GBuffer」要改读为「写 Substrate bitstream + 后续分类解算」两段式。

---

## 3. MegaLights 导读卡

| 栏目 | 内容 |
|------|------|
| **是什么** | 新一代直接光照路径：把「每盏灯一次屏幕空间 draw/dispatch」改为「按像素随机采样灯光子集」的统一采样器，配合追踪数据（HWRT/SWRT）做阴影 |
| **何时需要关注** | ① 项目显式开启；② 场景灯数量极大（成百上千盏局部光）时评估性能方案；③ 读 `RenderLights` 时发现一半灯「不见了」（被划进 MegaLights 区间） |
| **依赖哪些数据** | `FSortedLightSetSceneInfo` 中的 MegaLights 区间；追踪数据（`HasRequiredTracingData`） |
| **数据从哪来** | `GatherAndSortLights`（LightRendering.cpp:1409）给每盏灯算 `bHandledByMegaLights`（:1483）并划出 `MegaLightsLightStart`（:1581） |
| **何时执行** | `FDeferredShadingSceneRenderer::RenderMegaLights`（MegaLights/MegaLights.cpp:2428），调用点 DeferredShadingRenderer.cpp:3471，紧挨 `RenderLights`（:3467）之后 |

### 3.1 开关判定链（5.8 实测默认值）

```
MegaLights::IsEnabled()                       MegaLights.cpp:546
 = IsRequested() (:531) && HasRequiredTracingData() (:541)
     │
     ├─ r.MegaLights.Supported = 1        编译级支持 (ReadOnly, :19)
     ├─ r.MegaLights.EnableForProject = 0 项目总闸（默认关，注释: Experimental）
     ├─ r.MegaLights.Allowed = 1          scalability 允许
     ├─ PPV.FinalPostProcessSettings.bMegaLights（后处理体积开关，Scene.h:1887）
     └─ HasRequiredTracingData: 需要 HWRT 或 SWRT 追踪数据
```

⚠️ **限制**：MegaLights **不支持 Directional Light**——主平行光永远走传统 `RenderLights` 路径。这也是为什么两条路径必须共存。

### 3.2 与旧 RenderLights 的分工

```
GatherAndSortLights (LightRendering.cpp:1409)
   │  按 (bHandledByMegaLights, 类型, ...) 排序，得到两个区间边界
   ▼
FSortedLightSetSceneInfo
   ├─ [UnbatchedLightStart, MegaLightsLightStart)  → RenderLights        传统逐灯
   └─ [MegaLightsLightStart, Num)                  → RenderMegaLights    统一采样
```

主线判定：`SortedLightSet.MegaLightsLightStart < SortedLights.Num()`（DeferredShadingRenderer.cpp:3023）决定是否创建 `FMegaLightsFrameTemporaries`。**关闭时所有灯都在传统区间，RenderLights 就是全部**。

---

## 4. 对照：两套新架构对「读主线」的影响

| 维度 | Substrate | MegaLights |
|------|-----------|------------|
| 关闭时对主线的影响 | 零（全部短路） | 零（所有灯走传统路径） |
| 开启后改变什么 | GBuffer 布局 + BasePass 后多一组分类 Pass | 灯光循环换成采样 Pass |
| 数据源头 | 材质编译产物 | GatherAndSortLights 的排序结果 |
| 需要的前置知识 | BSDF/材质模型、GBuffer 编码 | 蒙特卡洛采样、RT 阴影 |
| 第一遍阅读决策 | **跳过**，记住 5 个分叉点位置 | **跳过**，记住区间划分逻辑 |

---

## 5. 小结

| 要点 | 说明 |
|------|------|
| Substrate | 材质/BSDF 重构；`r.Substrate` 默认 0；分叉点在 BasePass 及 :3168 分类段 |
| MegaLights | 直接光照重构；`EnableForProject` 默认 0 且 Experimental；不支持平行光 |
| 共存机制 | 灯光排序划出两个区间，传统与新路径分管 |
| 阅读策略 | 第一遍双双跳过，但要在主线上认得它们的调用点（见 05 篇主线表） |
| 深入入口 | Substrate：`Substrate/Substrate.cpp` + 材质侧 `SubstrateMaterial.cpp`；MegaLights：`MegaLights/MegaLights.cpp:2428` |

---

*上一篇：[02-核心概念Primitive与MeshDrawCommand](02-核心概念Primitive与MeshDrawCommand.md) | 下一篇：[04-渲染数据的来源与组织](04-渲染数据的来源与组织.md)*
