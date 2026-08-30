# 游戏性能分析 — GPU 性能分析

> 本文档是「游戏性能分析」系列文档的第 **2** 篇，详细阐述 GPU 侧性能剖析方法与工具实践，涵盖帧捕获与检视原理、RenderDoc 与 NVIDIA NSight Graphics 的深度使用、GPU 计数器分析、Shader 性能剖析以及常见 GPU 瓶颈模式。

---

## 1. GPU 性能分析概述

### 1.1 GPU 分析与 CPU 分析的本质差异

CPU 分析关注"时间花在哪个函数"，GPU 分析关注"一帧的渲染管线在哪里卡住"：

```
CPU 分析视角:
  函数A 3ms → 函数B 2ms → 函数C 5ms → 总计 10ms
  └─ 找到最慢的函数, 优化它

GPU 分析视角:
  ┌──────┐   ┌───────┐   ┌──────┐   ┌──────┐
  │顶点   │──→│光栅化  │──→│片元   │──→│输出  │
  │着色器 │   │       │   │着色器 │   │合并  │
  └──┬───┘   └───┬───┘   └──┬───┘   └──┬───┘
     │           │           │           │
   顶点不足?   三角形小?   Overdraw?  带宽?
   └─ 需要判断管线哪个阶段是瓶颈
```

### 1.2 GPU 管线瓶颈分层

GPU 渲染管线的性能瓶颈可归类为三大层级：

```
┌─────────────────────────────────────────────────────────────┐
│                    GPU 渲染管线瓶颈分类                       │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  1. 应用阶段瓶颈 (Application Stage)                  │  │
│  │     → Draw Call 数量过多、状态切换频繁                  │  │
│  │     → 通常是 CPU 提交问题 (参考 01-CPU分析)            │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  2. 几何阶段瓶颈 (Geometry Stage)                      │  │
│  │     → 顶点着色器过重、曲面细分过度、几何放大严重         │  │
│  │     → 顶点数量远超屏幕可见像素                          │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  3. 光栅化阶段瓶颈 (Rasterizer Stage)                 │  │
│  │     → 三角形过小导致光栅化效率低                        │  │
│  │     → Overdraw 严重 (重叠绘制)                          │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  4. 像素阶段瓶颈 (Pixel/Fragment Stage)                │  │
│  │     → 片元着色器指令过重、纹理采样过多、分支发散          │  │
│  │     → 分辨率过高、后处理 Pass 叠加                     │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  5. 带宽瓶颈 (Memory Bandwidth)                       │  │
│  │     → GBuffer 读写、纹理带宽、RT 切换                   │  │
│  │     → 移动端 TBDR 带宽是首要约束                        │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 1.3 GPU 瓶颈快速判断法

无需工具即可初步判断的实验方法：

| 实验 | 操作 | 如果帧时间不变 | 说明 |
|------|------|--------------|------|
| **降分辨率** | 渲染分辨率 ×0.5 | → **像素阶段瓶颈** | 像素数量减半但帧时间不变 = 像素处理不是瓶颈 |
| | | 帧时间下降 | → 像素阶段确实是瓶颈 |
| **简化 Shader** | 用最简 Shader 替换 | 帧时间不变 | → 非 Shader 瓶颈（可能是几何/带宽） |
| | | 帧时间下降 | → Shader 确实是瓶颈 |
| **减少顶点** | LOD 降低顶点数 | 帧时间不变 | → 非几何瓶颈 |
| | | 帧时间下降 | → 几何阶段瓶颈 |
| **减少纹理** | 用 1×1 纯色纹理 | 帧时间下降 | → 带宽/纹理采样瓶颈 |

> **原则**：这些实验法是"快速初筛"，精确定位仍需 GPU 分析工具。但它们能帮助你在打开工具前就缩小排查范围。

### 1.4 关键指标速查

GPU 剖析工具（NSight Graphics / PIX / Snapdragon Profiler / RenderDoc）会暴露大量计数器，初学者容易被淹没。下面按"你试图回答什么问题"来分组，只列出**最常用来做判断的指标**。

#### 1.4.1 时间类指标 — "GPU 的时间花在哪个 Pass / Draw Call"

| 指标 | 含义 | 怎么看 | 工具来源 |
|------|------|--------|---------|
| **GPU Frame Time** | 一帧从 GPU 开始执行到完成的总时间 | 与 CPU Frame Time 对比，判断 CPU-bound 还是 GPU-bound | NSight Graphics / PIX / stat gpu (UE) |
| **GPU Time per Draw Call** | 单个 Draw Call 的 GPU 执行耗时 | 按 GPU Time 排序，找最慢的 Top-10 Draw Call | NSight Graphics Range Profiler / RenderDoc |
| **GPU Time per Pass** | 一个渲染 Pass（如 Shadow / Base / Transparent）的总 GPU 耗时 | 按 Pass 聚合，判断哪个 Pass 最贵 | NSight Graphics / UE GPU Visualizer |
| **CPU-GPU Sync Time** | GPU 等待 CPU 提交命令的空闲时间 | Sync 高 → CPU 提交太慢，GPU 在空等 | NSight Systems / Unreal Insights |

```
GPU Frame Time 拆解示例 (NSight Graphics):

  ┌──────────────────────────────────────────────────────────┐
  │  Frame Total: 18.2ms                                     │
  │                                                          │
  │  ├─ Shadow Pass:       2.1ms  (12%)  ← 可接受            │
  │  ├─ Z-Prepass:         0.8ms  ( 4%)                      │
  │  ├─ Base Pass:         7.5ms  (41%)  ← 最大占比, 重点     │
  │  │  └─ 最慢 Draw: 角色皮肤 PS  3.2ms                     │
  │  ├─ Transparent Pass:  3.0ms  (17%)  ← Overdraw 嫌疑     │
  │  ├─ Post-Process:      3.8ms  (21%)  ← Bloom + DOF       │
  │  └─ UI:                1.0ms  ( 5%)                      │
  └──────────────────────────────────────────────────────────┘
```

#### 1.4.2 利用率与吞吐类指标 — "GPU 硬件单元在忙还是在闲"

这是判断 GPU 瓶颈归属的核心指标组。

| 指标 | 含义 | 正常范围 | 异常信号 | 告诉你什么 |
|------|------|---------|---------|-----------|
| **SM Active / SM Utilization** | 有活跃 Warp 的 SM 占比 | > 70% | < 50% → GPU 算力未充分利用（Draw Call 太少 / 工作量不均） | GPU 是否在"干活" |
| **Warp Occupancy (占用率)** | 每 SM 上同时驻留的 Warp 数 / 最大 Warp 数 | > 50% | < 25% → 寄存器/共享内存使用过多，或线程块太小 | 并行度是否足够隐藏延迟 |
| **Triangle Throughput** | 每秒处理的三角形数 (MTris/s) | 接近 GPU 峰值 | 远低于峰值 → 几何阶段不是瓶颈，或三角形太小 | 前端是否饱和 |
| **Fill Rate (像素填充率)** | 每秒写入的像素数 (GPixel/s) | 接近 GPU 峰值 | 远低于峰值 → 后端不是瓶颈 | 后端是否饱和 |
| **Compute Throughput (FLOPS)** | Compute Shader 的浮点运算吞吐 | — | ALU 利用率低 → Compute Shader 受限于内存带宽而非计算 | Compute 是 ALU-bound 还是 Bandwidth-bound |
| **Texture Throughput** | 每秒纹理采样操作数 | — | 低 → 纹理采样不是瓶颈 | TMU 是否饱和 |

```
SM Utilization vs Warp Occupancy 的区别:

  SM Utilization = "有多少 SM 在工作"
     └─ 低 → GPU 整体没喂饱（Draw Call 少 / 工作量小）

  Warp Occupancy = "每个 SM 里塞了多少 Warp"
     └─ 低 → 单个 SM 内并行度不够，无法隐藏内存延迟
     └─ 常见原因: 寄存器用量大 / 线程块太小 / 共享内存占用过多

  ┌───────────────────────────────────────────────────────┐
  │  SM Active 90% + Occupancy 80% → GPU 充分利用          │
  │  SM Active 90% + Occupancy 20% → SM 在跑但效率低       │
  │    → 可能 Memory Stall（Warp 发出访存后全部卡住）       │
  │  SM Active 30% → GPU 整体没喂饱                        │
  │    → Draw Call 太少 / 工作量太小                        │
  └───────────────────────────────────────────────────────┘
```

#### 1.4.3 Stall 类指标 — "GPU 在等什么"

这是你提到的 "stall 率"，是最关键的 GPU 微架构诊断指标之一。

| 指标 | 含义 | 异常信号 | 告诉你什么 |
|------|------|---------|-----------|
| **Warp Stall Rate** | Warp 因等待而停顿的周期占比 | > 50% → GPU 大量时间在等待而非计算 | 需要进一步看 Stall Reason |
| **Stall: Memory** | 等待显存/缓存返回数据 | 高 → 纹理采样 / Buffer 读取未命中 | 纹理太大 / 访问模式分散 / 缓存不友好 |
| **Stall: Sync** | 等待同步（Barrier / Fence / Atomic） | 高 → 过多的 GPU 同步点 | 减少 Pass 间依赖 / 合并 Pass |
| **Stall: Execution (依赖)** | 等待前一条指令的结果 | 高 → 长延迟指令链（如三角函数 / 递归纹理采样） | 减少依赖链 / 增加独立计算 |
| **Stall: Texture** | 等待纹理采样返回 | 高 → 纹理采样是瓶颈 | 降纹理分辨率 / 改采样模式 / 用 Mipmap |
| **Stall: Pipe (流水线)** | 等待其他管线阶段的输出 | 高 → 管线阶段间不均衡 | 如 VS 太重导致 PS 空等 |

```
Warp Stall 分析思路 (NSight Graphics):

  Warp Stall Rate = 65%
  ├─ Memory Stall:     40%  ← 主要原因: 等显存
  │  └─ 可能: 纹理太大 / GBuffer 带宽 / 随机访问
  │  └─ 优化方向: 压缩纹理 / 减少纹理采样数 / 改用计算友好布局
  │
  ├─ Sync Stall:       15%  ← Barrier / Fence
  │  └─ 可能: Pass 间依赖太多 / UAV 读写后立即读
  │  └─ 优化方向: 合并 Pass / 延迟读取
  │
  └─ Execution Stall:  10%  ← 指令依赖
     └─ 可能: Shader 中大量连续的 sin/cos/pow
     └─ 优化方向: 预计算到查找表 / 减少指令数
```

> **关键理解**：GPU 的 Stall 率高 ≠ "代码写得差"。GPU 的 SIMT 模型本质上是靠大量 Warp 并行来隐藏延迟的。如果 Occupancy 足够高（有足够的 Warp 可以切换），即使单 Warp Stall 严重，整体吞吐也不会下降。**只有当 Stall Rate 高 + Occupancy 也低时，才是真正的性能问题。**

#### 1.4.4 带宽类指标 — "显存读写是不是瓶颈"

| 指标 | 含义 | 正常范围 | 异常信号 | 告诉你什么 |
|------|------|---------|---------|-----------|
| **Memory Bandwidth Utilization** | 实际带宽 / GPU 峰值带宽 | < 80% | > 90% → Bandwidth-bound | 显存带宽已饱和，再多的计算也无法提速 |
| **L1 Cache Hit Rate (纹理/计算)** | L1 缓存命中率 | > 70% | < 40% → 纹理访问模式不缓存友好 | 纹理布局 / UV 连续性差 |
| **L2 Cache Hit Rate** | L2 缓存命中率 | > 60% | < 30% → 工作集超出 L2 | 减少同时使用的纹理 / 分块渲染 |
| **VRAM Read/Write (GB/s)** | 实际读取/写入速率 | — | 写入远大于读取 → Overdraw 严重 | 检查透明排序 / Z-Prepass |
| **Texture Bandwidth** | 纹理采样占用的带宽 | — | 占总带宽 > 60% → 纹理是带宽主因 | 压缩纹理 / Mipmap / 减少采样数 |

```
Bandwidth-bound 判断:

  Bandwidth Utilization > 90% + SM Active < 60%
  → GPU 被内存带宽限制，计算单元在等数据
  
  常见于:
  ├─ 延迟渲染的 GBuffer 读写 (4-8 个 RT)
  ├─ 高分辨率 + 大量纹理采样
  ├─ 移动端 TBDR 的 Tile Memory 溢出 (回退到 VRAM 往返)
  └─ 后处理链太长 (每步都全屏读写)
```

#### 1.4.5 管线阶段占比类指标 — "前端还是后端"

| 指标 | 含义 | 异常信号 | 告诉你什么 |
|------|------|---------|-----------|
| **Vertex Processing %** | 顶点着色器 + 曲面细分占 GPU 时间比例 | > 40% → 几何阶段瓶颈 | 减三角形 / 简化 VS / 关曲面细分 |
| **Rasterization %** | 光栅化阶段占比 | > 20% → 三角形太小或太多 | LOD / 合并小三角形 |
| **Pixel/Fragment Shader %** | 片元着色器占比 | > 50% → 像素着色器瓶颈 | 简化 PS / 降分辨率 / 减少光照计算 |
| **ROP / Output %** | 输出合并阶段占比 | > 15% → 混合/深度测试开销大 | 减少透明物体 / Z-Prepass |
| **Compute %** | Compute Shader 占比 | — | Lumen / 粒子 / 后处理计算是否过重 |

```
管线阶段占比 (NSight Graphics GPU Pipeline View):

  ┌───────────────────────────────────────────────────────┐
  │  Frame GPU Time: 18ms                                 │
  │                                                      │
  │  Vertex:     ██████ 3.2ms  (18%)  ← 可接受            │
  │  Raster:     ██ 1.0ms  ( 6%)                         │
  │  Pixel:      ████████████████ 9.5ms  (53%)  ← 瓶颈!  │
  │  ROP:        ███ 1.8ms  (10%)                        │
  │  Compute:    ██ 2.5ms  (14%)                         │
  │                                                      │
  │  结论: Pixel Shader 是主要瓶颈                        │
  │  → 降分辨率 / 简化 PS / 减少光源数                    │
  └───────────────────────────────────────────────────────┘
```

#### 1.4.6 指标 → 瓶颈类型速查

```
你看到的指标异常                       →   GPU 瓶颈类型

SM Active 低 + Occupancy 低           →   GPU 没喂饱 (Draw Call 少 / 工作量小)
SM Active 高 + Occupancy 低 + Stall 高 →   Memory-bound (带宽/缓存未命中)
SM Active 高 + Occupancy 高            →   GPU 充分利用，需看阶段占比
Pixel Shader % > 50%                   →   像素瓶颈 (降分辨率 / 简化 PS)
Vertex Processing % > 40%              →   几何瓶颈 (LOD / 减三角形)
Bandwidth Utilization > 90%            →   带宽瓶颈 (压缩纹理 / 减 RT / 合并 Pass)
Warp Stall: Sync 高                    →   同步瓶颈 (减 Barrier / 合并 Pass)
Warp Stall: Texture 高 + L1 Hit 低     →   纹理缓存差 (改布局 / Mipmap / 降分辨率)
Fill Rate 接近峰值                     →   填充率瓶颈 (减 Overdraw / 降分辨率)
Triangle Throughput 接近峰值           →   几何吞吐瓶颈 (LOD / 减三角形)
```

---

## 2. 帧捕获与检视原理

### 2.1 什么是帧捕获

GPU 帧捕获工具的核心能力是**录制一帧完整的渲染操作序列**，然后逐条回放和检视：

```
帧捕获原理:
┌──────────────────────────────────────────────────────────┐
│  实时渲染 (运行中)                                        │
│  Frame N:                                                │
│    vkCmdBeginRenderPass()                                │
│    vkCmdBindPipeline()                                  │
│    vkCmdBindDescriptorSets()                             │
│    vkCmdDrawIndexed()  ← Draw Call #1                    │
│    vkCmdDrawIndexed()  ← Draw Call #2                    │
│    vkCmdBindPipeline()  ← 状态切换                        │
│    vkCmdDrawIndexed()  ← Draw Call #3                    │
│    ...                                                   │
│    vkCmdEndRenderPass()                                 │
│  Frame N+1: ...                                          │
│                                                          │
│  捕获点: 在 vkQueueSubmit 或 Present 前插入               │
│  → 记录所有 API 调用、参数、资源状态                       │
│  → 快照所有 GPU 资源 (纹理/Buffer/RT)                    │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│  帧回放 (分析中)                                          │
│  Event Browser:                                          │
│  ├─ #0 vkCmdBeginRenderPass                              │
│  ├─ #1 vkCmdBindPipeline (Graphics)                      │
│  ├─ #2 vkCmdBindDescriptorSets                           │
│  ├─ #3 vkCmdDrawIndexed ← 选中此 Draw Call               │
│  │   → 查看绑定的顶点/索引 Buffer                        │
│  │   → 查看绑定的纹理和采样器                            │
│  │   → 查看着色器源码和反汇编                            │
│  │   → 查看此 Draw Call 前后的 RT 状态                    │
│  │   → 查看此 Draw Call 的像素输出                        │
│  ├─ #4 vkCmdDrawIndexed                                  │
│  └─ ...                                                  │
└──────────────────────────────────────────────────────────┘
```

### 2.2 捕获数据的内容

一次帧捕获包含以下完整数据：

| 数据类别 | 内容 | 分析用途 |
|---------|------|---------|
| **API 调用序列** | 所有 vkCmd*/gl*/ID3D 命令及参数 | 逐条检视 Draw Call、状态切换 |
| **管线状态** | Pipeline、RenderPass、BlendState 等 | 诊断状态错误、冗余切换 |
| **着色器** | 源码 + 反汇编 + 寄存器分配 | Shader 性能分析、指令计数 |
| **纹理资源** | 所有绑定的纹理内容与格式 | 纹理格式/大小/压缩检查 |
| **Buffer 数据** | 顶点/索引/Uniform Buffer 内容 | 几何数据验证、数据布局检查 |
| **渲染目标** | 每个 Pass 的 RT 前后状态 | Overdraw 可视化、像素历史追踪 |
| **时间戳** | 各 Draw Call/Pass 的 GPU 耗时 | 精确帧时间拆分 |

---

## 3. RenderDoc

### 3.1 RenderDoc 定位

RenderDoc 是最流行的**免费开源** GPU 帧调试工具，支持 Vulkan、D3D11/12、OpenGL：

```
RenderDoc 核心能力:
┌───────────────────────────────────────────────┐
│  帧捕获与回放                                   │
│  ├─ 捕获完整 API 调用序列和资源                  │
│  ├─ 逐条 Event 检视                             │
│  └─ 时间线浏览                                  │
│                                                │
│  资源检视                                       │
│  ├─ 纹理查看器 (格式/尺寸/Mipmap)               │
│  ├─ Buffer 查看器 (顶点/索引/结构化数据)        │
│  ├─ Pipeline State 查看器                       │
│  └─ Shader 查看器 (源码+反汇编)                │
│                                                │
│  像素级调试                                     │
│  ├─ Texture Viewer 的像素历史                   │
│  ├─ Pixel History (追踪单像素所有修改)          │
│  └─ Debug Overlay (Overdraw/Draw Call高亮)     │
│                                                │
│  限制                                          │
│  ├─ 不提供 GPU 性能计数器 (非剖析工具)          │
│  ├─ 不提供微架构级性能数据                      │
│  └─ 捕获开销较大, 不适合持续监控               │
└───────────────────────────────────────────────┘
```

> **关键纠正**：RenderDoc 是**帧调试工具**，不是性能剖析工具。它能告诉你"渲染了什么"和"状态对不对"，但不能告诉你"GPU 为什么慢"。性能剖析需要 NSight Graphics 或 GPU 计数器工具。

### 3.2 RenderDoc 工作流

```
捕获流程:
1. 启动 RenderDoc → 配置目标程序
   ├─ 可执行文件路径
   ├─ 工作目录
   ├─ 命令行参数
   └─ API (Vulkan/D3D12/D3D11/OpenGL)

2. 点击 "Launch" → RenderDoc 启动游戏
3. 进入目标场景, 等稳定
4. 按 F12 或点击 "Capture" → 捕获当前帧
5. 切回 RenderDoc → 打开捕获文件

分析流程:
┌─────────────────────────────────────────────────────┐
│ Event Browser (左侧)                                │
│ ├─ vkCmdBeginRenderPass(#0)                        │
│ │  └─ RT: GBuffer (Albedo/Normal/Roughness)        │
│ ├─ vkCmdBindPipeline(#1)                           │
│ ├─ vkCmdDrawIndexed(#2)  ← 选中                    │
│ │  └─ 选中后右侧更新                                │
│ ├─ vkCmdDrawIndexed(#3)                            │
│ ├─ vkCmdBindPipeline(#4)  ← 状态切换               │
│ └─ ...                                             │
│                                                     │
│ Pipeline State (右侧上方)                           │
│ ├─ Vertex Input: Buffer绑定、步距、属性             │
│ ├─ Shader Stages: VS/TES/TFS/FS 源码与反汇编       │
│ ├─ Rasterizer: 多边形模式、剔除、线宽               │
│ ├─ Color Blend: 混合方程、目标RT                    │
│ └─ Dynamic State: Viewport/Scissor                │
│                                                     │
│ Texture Viewer (右侧下方)                           │
│ ├─ Output: 当前RT (选中Draw Call后的状态)          │
│ ├─ Inputs: 绑定的纹理                               │
│ ├─ Depth: 深度缓冲状态                              │
│ └─ History: 像素历史追踪                             │
└─────────────────────────────────────────────────────┘
```

### 3.3 关键分析场景

**场景1：诊断 Overdraw**

```
RenderDoc → Texture Viewer → Output RT
├─ 右键 → "Show Overlay" → "Draw Call"
│   → 每个像素的 Draw Call 次数可视化
│   ├─ 黑色 = 0次
│   ├─ 蓝色 = 1次 (正常)
│   ├─ 绿色 = 2-3次 (可接受)
│   ├─ 黄色 = 4-7次 (偏高)
│   └─ 红色 = 8+次 (严重 Overdraw)
│
├─ 定位红色区域 → 查看是哪些 Draw Call
├─ 分析是否可以:
│   ├─ 添加 Early-Z 预通道
│   ├─ 前→后排序 (Z-Pre-Pass + Early-Z)
│   ├─ 合并透明层
│   └─ 使用 LOD 减少远距离绘制
```

**场景2：诊断纹理格式与大小**

```
Texture Viewer → 选中纹理 → 查看属性:
┌─────────────────────────────────────────┐
│ Texture: character_diffuse.dds          │
│ Format: R8G8B8A8_UNORM  ← 未压缩!      │
│ Size: 4096×4096                        │
│ Mipmaps: 12 levels                     │
│ Memory: 64 MB  ← 过大                  │
│                                         │
│ → 优化: 改用 BC7压缩 = 8 MB (省87%)    │
│ → 优化: 远距离物体用 2048 = 4 MB      │
└─────────────────────────────────────────┘
```

**场景3：诊断状态冗余切换**

```
Event Browser 中检查:
├─ #1 vkCmdBindPipeline(PipelineA)
├─ #2 vkCmdDrawIndexed()    ← 使用 PipelineA
├─ #3 vkCmdBindPipeline(PipelineB)  ← 切换
├─ #4 vkCmdDrawIndexed()    ← 使用 PipelineB
├─ #5 vkCmdBindPipeline(PipelineA)  ← 又切回! 冗余!
├─ #6 vkCmdDrawIndexed()
│
→ 优化: 按 Pipeline 排序 Draw Call
   PipelineA 的所有 Draw → PipelineB 的所有 Draw
   减少 Pipeline 切换次数
```

### 3.4 Shader 调试

RenderDoc 支持在 Shader 中设置断点，逐行调试：

```
Shader 查看器 → 选中片元着色器
├─ 源码视图: 高亮当前执行的行
├─ 反汇编视图: SPIR-V / DXIL 反汇编
├─ 变量面板: 查看 uniform 和局部变量
│
│ 调试操作:
├─ 在 Texture Viewer 中选择像素
├─ 右键 → "Debug Pixel" → 进入 Shader 调试
├─ 逐行执行, 查看变量值变化
├─ 定位 Shader 逻辑错误
│
│ 性能提示 (非精确):
├─ 反汇编中指令数量 → Shader 复杂度估计
├─ 纹理采样次数 → 带宽估计
└─ 分支指令 → 发散风险估计
```

---

## 4. NVIDIA Nsight Graphics

### 4.1 NSight Graphics 定位

与 RenderDoc 不同，Nsight Graphics 是**GPU 性能剖析工具**，提供硬件级性能计数器：

```
RenderDoc vs NSight Graphics 对比:
┌────────────────────┬───────────────────┬──────────────────────┐
│ 维度               │ RenderDoc         │ Nsight Graphics       │
├────────────────────┼───────────────────┼──────────────────────┤
│ 核心定位           │ 帧调试            │ 性能剖析              │
│ API 调用检视       │ ✅ 逐条           │ ✅ 逐条               │
│ 资源/纹理查看      │ ✅                │ ✅                    │
│ Shader 调试        │ ✅ 逐行断点       │ ✅ (有限)             │
│ GPU 性能计数器     │ ❌                │ ✅ 硬件级             │
│ 帧时间精确拆分     │ ❌                │ ✅ 每个 Draw Call     │
│ Shader 性能分析    │ ❌                │ ✅ 指令级             │
│ 瓶颈分类           │ ❌                │ ✅ 几何/像素/带宽      │
│ GPU 架构           │ 跨平台            │ NVIDIA 专属           │
│ 适用场景           │ "渲染了什么"      │ "GPU 为什么慢"        │
└────────────────────┴───────────────────┴──────────────────────┘
```

### 4.2 NSight Graphics 分析模式

```
NSight Graphics 分析类型:
├─ Frame Debugger (帧调试)
│  ├─ 类似 RenderDoc 的逐 Draw Call 检视
│  └─ 但额外提供每个 Draw Call 的 GPU 耗时
│
├─ Range Profiler (范围剖析)  ← 核心功能
│  ├─ 选择一组 Draw Call 作为"范围"
│  ├─ 深度分析该范围的 GPU 性能
│  └─ 输出瓶颈分类 + 计数器 + Shader 分析
│
├─ GPU Trace (GPU 追踪)
│  ├─ 全帧 GPU 时间线
│  └─ 各 Queue / Pass / Draw Call 的时间分布
│
└─ Counter Explorer (计数器浏览)
   └─ 实时查看任意 GPU 计数器
```

### 4.3 Range Profiler 深度解读

Range Profiler 是 NSight Graphics 最强大的功能，它对选定范围的 Draw Call 进行**瓶颈分类分析**：

```
Range Profiler 输出示意:
┌─────────────────────────────────────────────────────────────┐
│  Range: Draw Call #15-#42 (场景主渲染)                       │
│  Duration: 8.5ms                                             │
│                                                             │
│  瓶颈分类 (GPU 利用率):                                      │
│  ┌─────────────────────────────────────────────┐            │
│  │  ██████████████████████████  65%  像素着色器  │ ← 瓶颈!  │
│  │  ████████████████          35%  几何/顶点     │            │
│  │  ████                        8%  光栅化       │            │
│  │  ██                          3%  带宽         │            │
│  └─────────────────────────────────────────────┘            │
│  → 像素着色器占 65%, 是该范围的主要瓶颈                      │
│                                                             │
│  关键计数器:                                                 │
│  ├─ Pixel/Fragment Throughput: 45 Gpix/s (峰值 60 Gpix/s)  │
│  ├─ Vertex Throughput:        12 Mvert/s                    │
│  ├─ Texture Bandwidth:        180 GB/s (峰值 400 GB/s)     │
│  ├─ L2 Cache Hit Rate:        72%                           │
│  ├─ SM Occupancy:             65% (理想 >75%)              │
│  └─ Warp Divergence:          18% (理想 <5%)               │
│                                                             │
│  Shader 性能 (最慢 Shader):                                 │
│  ├─ Shader: CharacterForward.frag                           │
│  ├─ 指令数: 342 (ALU: 180, TEX: 28, 控制流: 15)            │
│  ├─ 纹理采样: 8 次/像素                                     │
│  ├─ 寄存器使用: 48/255 (影响 Occupancy)                     │
│  └─ 分支发散: 18% (if/else 导致 Warp 发散)                  │
│                                                             │
│  优化建议:                                                   │
│  ├─ 像素着色器指令过多 → 简化计算/预计算                    │
│  ├─ 纹理采样 8 次 → 减少采样或使用 Mipmap                   │
│  └─ 分支发散 18% → 消除 if/else 分支                        │
└─────────────────────────────────────────────────────────────┘
```

### 4.4 GPU 计数器详解

| 计数器类别 | 关键计数器 | 含义 | 异常阈值 |
|-----------|-----------|------|---------|
| **吞吐量** | Pixels/Sec | 像素填充率 | 接近峰值 = 像素瓶颈 |
| | Vertices/Sec | 顶点处理率 | 接近峰值 = 几何瓶颈 |
| | Triangles/Sec | 三角形处理率 | 接近峰值 = 几何瓶颈 |
| **带宽** | Texture Bandwidth | 纹理带宽 | >70% 峰值 = 带宽瓶颈 |
| | Color Bandwidth | 色彩附件读写 | >70% 峰值 = 带宽瓶颈 |
| | Depth Bandwidth | 深度读写 | 偏高 = Overdraw |
| **缓存** | L1 Cache Hit Rate | L1 命中率 | <80% = 数据局部性差 |
| | L2 Cache Hit Rate | L2 命中率 | <60% = 纹理/RT 复用差 |
| | Texture Cache Hit | 纹理缓存命中 | <70% = 纹理布局差 |
| **占用率** | SM Occupancy | Shader 多处理器占用 | <50% = 低效 |
| | Warp Occupancy | Warp 活跃比 | <60% = 低效 |
| | Register Usage | 寄存器使用量 | >128 = 影响 Occupancy |
| **发散** | Warp Divergence | Warp 分支发散率 | >10% = 分支问题 |

### 4.5 Shader 性能分析

NSight Graphics 的 Shader 性能分析提供指令级洞察：

```
Shader 反汇编分析 (NSight Graphics):
┌───────────────────────────────────────────────────────┐
│ CharacterForward.frag - 反汇编分析                     │
│                                                       │
│ 指令分布:                                              │
│ ├─ ALU (算术逻辑):    180 条 (52.6%)                  │
│ ├─ TEX (纹理采样):     28 条  (8.2%) ← 8次实际采样   │
│ ├─ Flow (控制流):      15 条  (4.4%) ← 分支来源      │
│ ├─ LD/ST (加载存储):   12 条  (3.5%)                 │
│ └─ 其他:               107 条 (31.3%)                │
│ 总计: 342 条指令                                      │
│                                                       │
│ 寄存器分析:                                           │
│ ├─ 使用寄存器: 48 / 255 (GPR)                         │
│ ├─ → 48 GPR 限制 Occupancy 至 75%                    │
│ ├─ 如果降至 32 GPR → Occupancy 提升至 100%           │
│ └─ 每个 Warp 需要更多寄存器 = 同时活跃 Warp 更少      │
│                                                       │
│ 纹理采样分析:                                         │
│ ├─ 采样 1: diffuse_map    (BC7压缩, Mip线性)          │
│ ├─ 采样 2: normal_map     (BC5压缩, Mip线性)          │
│ ├─ 采样 3: roughness_map  (BC4压缩)                   │
│ ├─ 采样 4: ao_map         (BC4压缩)                   │
│ ├─ 采样 5: shadow_map     (PCF 3×3 = 9次实际采样!)    │
│ ├─ 采样 6: irradiance_SH  (无过滤)                    │
│ ├─ 采样 7: prefiltered_env (Mip三线性)                │
│ └─ 采样 8: brdf_lut       (无过滤)                   │
│ → 8次逻辑采样, 但阴影 PCF 实际 16次硬件采样           │
│ → 带宽消耗集中在阴影采样                              │
│                                                       │
│ 分支发散分析:                                         │
│ ├─ if (isMetallic) { ... } else { ... }              │
│ │  → 同一 Warp 内部分像素 metallic=true, 部分=false  │
│ │  → 两分支都执行, 效率减半                           │
│ └─ 发散率: 18%                                        │
└───────────────────────────────────────────────────────┘
```

---

## 5. NVIDIA Nsight Systems

### 5.1 Nsight Systems 定位

Nsight Systems 是**系统级追踪工具**，关联 CPU 和 GPU 的时间线：

```
Nsight Systems 核心价值: CPU-GPU 时间线关联

时间线视图:
         0ms        5ms       10ms       15ms       20ms
CPU:     ┌──Game──┐┌─Render─┐┌──Wait──┐┌──Game──┐
         │  逻辑    ││  提交   ││ GPU完  ││  逻辑   │
         └─────────┘└────────┘│  成?   │└────────┘
                              └────────┘
GPU:                    ┌──Render──┐
                        │ GPU执行   │
                        │ 12ms      │
                        └───────────┘
                         ↑
                    CPU 等 GPU 完成
                    → 这里是同步等待

关键发现:
├─ CPU "Wait" 段 = CPU 在等待 GPU, 说明 CPU 快于 GPU
├─ → GPU-bound, 优化 GPU 侧
├─ 如果 GPU 有空闲段 = CPU 提交不够快
├─ → CPU-bound, 优化 CPU 提交
└─ 时间线可以显示多线程: Game Thread / Render Thread / Worker Threads
```

### 5.2 与 Nsight Graphics 的配合

```
分析流程:
Step 1: Nsight Systems
  → 获取 CPU+GPU 时间线
  → 判断 CPU-bound 还是 GPU-bound
  → 定位问题时间段

Step 2: 如果 GPU-bound → Nsight Graphics
  → 帧捕获, Range Profiler
  → 定位 GPU 瓶颈 (像素/几何/带宽)
  → Shader 级分析

Step 3: 如果 CPU-bound → VTune
  → 参考 01-CPU性能分析
  → 定位 CPU 热点函数
```

---

## 6. 其他 GPU 分析工具

### 6.1 PIX (Windows/Xbox)

```
PIX 定位:
├─ Microsoft 官方工具, D3D12 专属
├─ Windows + Xbox 平台
├─ 核心能力:
│  ├─ 帧捕获与回放 (类似 RenderDoc)
│  ├─ GPU 计时 (每个 Draw Call)
│  ├─ GPU 计数器 (基础)
│  └─ 内存分析
├─ 优势: D3D12 深度支持, Xbox 开发必需
└─ 劣势: 不支持 Vulkan/OpenGL
```

### 6.2 Xcode GPU Frame Debugger (Metal)

```
Xcode Metal 工具:
├─ macOS/iOS 专属, Metal API
├─ 核心能力:
│  ├─ GPU 帧捕获
│  ├─ Shader 调试 (Metal Shader 逐行)
│  ├─ 性能计数器 (Apple GPU)
│  └─ 能耗分析 (iOS)
├─ 优势: Apple 生态深度集成, 能耗分析独有
└─ 劣势: 仅限 Apple 平台
```

### 6.3 移动端 GPU 工具

| 工具 | GPU 厂商 | 平台 | 核心能力 |
|------|---------|------|---------|
| **Snapdragon Profiler** | Qualcomm Adreno | Android | GPU 计数器、帧捕获、功耗分析 |
| **Mali Graphics Debugger** | ARM Mali | Android | 帧调试、性能计数器 |
| **Mali Streamline** | ARM Mali | Android | 系统级性能+功耗追踪 |
| **PVRMonitor** | Imagination PowerVR | Android | GPU 计数器实时监控 |
| **GPU PerfStudio** | AMD Radeon | Windows | Radeon GPU 性能分析 |

---

## 7. GPU 瓶颈模式与优化方向

### 7.1 像素/片元瓶颈

```
症状:
├─ Range Profiler 显示像素着色器占比 >50%
├─ 降分辨率帧时间显著下降
├─ 纹理采样次数高

优化方向:
├─ 简化着色器: 合并计算、预计算查找表
├─ 减少纹理采样: 合并纹理通道 (ORM 贴图)
├─ 降低分辨率: 渲染分辨率 < 显示分辨率 + 升采样
├─ Early-Z: 前向排序 + 深度预通道
├─ 减少后处理 Pass: 合并后处理步骤
└─ 分辨率自适应: 动态分辨率缩放
```

### 7.2 几何/顶点瓶颈

```
症状:
├─ 顶点处理占比高
├─ 减少顶点数帧时间下降
├─ 曲面细分或几何着色器使用

优化方向:
├─ LOD: 远距离使用低多边形模型
├─ 遮挡剔除: 不渲染被遮挡的物体
├─ 视锥剔除: 不渲染视野外物体
├─ 减少曲面细分: 按距离调整 Tess Factor
├─ Mesh 优化: 顶点缓存优化 (FIFO 优化索引)
└─ 几何合并: 静态网格合批
```

### 7.3 带宽瓶颈

```
症状:
├─ 带宽计数器接近峰值
├─ 缓存命中率低
├─ 移动端最常见瓶颈 (TBDR 带宽预算紧张)

优化方向:
├─ 纹理压缩: BCn (PC) / ASTC (移动)
├─ GBuffer 精简: 减少渲染目标数量和位宽
├─ 资源别名: 不同 Pass 复用同一块显存
├─ Mipmap: 确保所有纹理有 Mipmap
├─ 减少后处理 RT 切换
├─ 移动端: 利用 Tile Memory (Subpass)
└─ 带宽排序: 带宽重的 Pass 放在分辨率低的阶段
```

### 7.4 Overdraw 问题

```
症状:
├─ 像素被多次绘制 (RenderDoc Draw Call 叠加图红色)
├─ 深度读写次数高
├─ 透明物体叠加严重

优化方向:
├─ Z-Pre-Pass: 先渲染深度, 再用 Early-Z 剔除
├─ 前向排序: 不透明物体从前到后排序
├─ 层次Z缓冲 (Hi-Z): GPU 遮挡剔除
├─ 减少透明物体: 合并透明层、使用 alpha 测试
├─ LOD: 远距离减少透明粒子
└─ 体积渲染: 用 Ray Marching 替代多层 Sprite
```

### 7.5 Wave/Warp 占用率低

```
症状:
├─ SM Occupancy < 50%
├─ 寄存器使用过多 (>128 GPR)
├─ 分支发散严重

优化方向:
├─ 减少寄存器: 简化 Shader、避免大量临时变量
├─ 消除分支: 数据驱动替代 if/else
│   // 差: if (metallic > 0.5) F = spec; else F = diff;
│   // 好: F = lerp(diff, spec, metallic);
├─ 分组排序: 按材质属性排序 Draw Call 减少发散
├─ 减少 Workgroup 大小不匹配
└─ 使用 Shader 编译器优化标志 (如 -O3)
```

---

## 8. 实战工作流：GPU 性能分析标准流程

```
Step 1: 确认 GPU-bound
├─ Nsight Systems 时间线 → GPU 执行 > CPU 提交
├─ 或引擎 Profiler → GPU 时间 > CPU 时间
└─ 确认后进入 GPU 分析

Step 2: 快速实验法初筛
├─ 降分辨率 → 像素瓶颈?
├─ 简化 Shader → Shader 瓶颈?
├─ 减少 LOD → 几何瓶颈?
└─ 缩小排查范围

Step 3: RenderDoc 帧检视
├─ 捕获帧 → 浏览 Event Browser
├─ 检查 Draw Call 数量和状态切换
├─ Texture Viewer → Overdraw 叠加图
├─ 检查纹理格式/大小/压缩
├─ 检查 Shader 绑定和源码
└─ 记录可疑的 Draw Call 范围

Step 4: NSight Graphics 深度剖析
├─ 帧捕获 → GPU Trace → 全帧时间分布
├─ Range Profiler → 选中最耗时范围
│   ├─ 瓶颈分类: 像素/几何/带宽?
│   ├─ 计数器分析: 哪些接近峰值?
│   └─ Shader 分析: 最慢 Shader 的指令/寄存器/采样
└─ 记录瓶颈类型和具体原因

Step 5: 针对性优化
├─ 像素瓶颈: 简化Shader/降分辨率/减少纹理采样
├─ 几何瓶颈: LOD/剔除/合批
├─ 带宽瓶颈: 纹理压缩/GBuffer精简/资源别名
├─ Overdraw: Z-Pre-Pass/排序/遮挡剔除
└─ 占用率低: 减少寄存器/消除分支

Step 6: 验证
├─ 相同场景重新捕获
├─ Range Profiler A/B 对比
├─ 确认瓶颈占比下降
└─ 检查是否引入新瓶颈 (优化A可能导致B变差)
```

---

## 9. 工具选择速查表

| 分析目标 | 首选工具 | 备选工具 | 核心指标 |
|---------|---------|---------|---------|
| Draw Call 检视 | RenderDoc | NSight Graphics | Draw Call 数、状态切换次数 |
| Overdraw 分析 | RenderDoc | NSight Graphics | 像素覆盖次数 |
| 纹理/Buffer 检视 | RenderDoc | NSight Graphics | 格式、大小、压缩 |
| Shader 调试 | RenderDoc | Xcode (Metal) | 逻辑错误定位 |
| GPU 计数器 | NSight Graphics | Snapdragon Profiler | 像素率、带宽、占用率 |
| 帧时间拆分 | NSight Graphics | PIX | 每 Draw Call 耗时 |
| CPU-GPU 关联 | Nsight Systems | Unreal Insights | 时间线对齐 |
| Shader 性能 | NSight Graphics | Xcode (Metal) | 指令数、寄存器、采样数 |
| 移动端 GPU | Snapdragon Profiler | Mali Graphics Debugger | 功耗、计数器 |

---

*上一篇：[01-CPU性能分析](01-CPU性能分析.md)*
*下一篇：[03-内存分析](03-内存分析.md)*
