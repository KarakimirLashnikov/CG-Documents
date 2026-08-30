# 游戏性能分析 — Unity 平台性能分析

> 本文档是「游戏性能分析」系列文档的第 **5** 篇，详细阐述 Unity 引擎的性能分析工具链，涵盖 Profiler 多维度剖析、Frame Debugger 逐 Draw Call 检视、Memory Profiler 内存分析、Profile Analyzer 帧对比，以及 Unity 特有的性能瓶颈模式与优化实践。

---

## 1. Unity 性能分析工具链概览

### 1.1 工具生态

```
Unity 性能分析工具矩阵:
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  ┌─────────────────┐  ┌──────────────────┐  ┌─────────┐│
│  │ Unity Profiler  │  │ Frame Debugger   │  │ Profile ││
│  │ (CPU/GPU/内存)  │  │ (Draw Call检视)   │  │ Analyzer││
│  └────────┬────────┘  └────────┬─────────┘  └────┬────┘│
│           │                     │                 │     │
│  ┌────────┴────────┐  ┌────────┴─────────┐  ┌───┴────┐│
│  │ Memory Profiler │  │ Rendering Stats   │  │ Device ││
│  │ (内存快照分析)   │  │ (渲染统计面板)    │  │ Profiler││
│  └─────────────────┘  └──────────────────┘  └────────┘│
│                                                          │
│  辅助工具:                                               │
│  ├─ Unity Performance Testing Framework (自动化基准)     │
│  ├─ Unity Cloud Performance Testing (云端回归)            │
│  ├─ Android Logcat / Xcode Instruments (原生层)          │
│  └─ RenderDoc / NSight Graphics (GPU 深度分析)           │
└──────────────────────────────────────────────────────────┘
```

### 1.2 工具选择指南

| 分析目标 | 首选工具 | 核心能力 |
|---------|---------|---------|
| CPU 帧时间拆分 | Unity Profiler (CPU Usage) | 按模块拆分帧时间 |
| GPU 帧时间拆分 | Unity Profiler (GPU Usage) | 各 Pass GPU 耗时 |
| Draw Call 分析 | Frame Debugger | 逐 Draw Call 状态检视 |
| Draw Call 数量 | Rendering Stats | 实时 Draw Call/三角形/批次统计 |
| 内存泄漏 | Memory Profiler | 快照对比、对象引用链 |
| 帧间对比 | Profile Analyzer | A/B 帧数据对比、P99 分位 |
| 移动端远程分析 | Device Profiler | 无线连接设备、实时监控 |
| 自动化回归 | Performance Testing Framework | 脚本驱动基准测试 |

---

## 2. Unity Profiler

### 2.1 Profiler 模块体系

Unity Profiler 提供**多模块并行剖析**，每个模块专注一个维度：

```
Unity Profiler 模块:
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  CPU Usage    → 帧时间按模块拆分 (最常用)                      │
│  ├─ Hierarchy 视图: 完整调用树                               │
│  ├─ Hierarchy 视图: 按类别聚合                                │
│  └─ Timeline 视图: 多线程时间线                               │
│                                                              │
│  GPU Usage   → GPU 各阶段耗时 (需启用 GPU Profiling)          │
│  ├─ Render.Mesh: 网格渲染                                     │
│  ├─ Render.OpaqueObjects: 不透明物体                          │
│  ├─ Render.TransparentObjects: 透明物体                       │
│  ├─ Camera.Render: 相机渲染总时间                             │
│  └─ PostProcessing: 后处理                                    │
│                                                              │
│  Memory     → 内存使用概览 (托管堆/原生/GPU)                  │
│  ├─ Total Allocated: 总分配                                   │
│  ├─ GC Alloc: 每帧 GC 分配 (→ 应为 0!)                       │
│  └─ Used Heap: 托管堆使用                                     │
│                                                              │
│  Rendering   → 渲染统计实时曲线                                │
│  ├─ Draw Calls                                               │
│  ├─ Batches / Saved by Batching                              │
│  ├─ Triangles / Vertices                                     │
│  └─ SetPass Calls (Shader 切换次数)                          │
│                                                              │
│  Audio      → 音频处理耗时                                    │
│  Physics    → 物理模拟耗时 (FixedUpdate)                     │
│  Physics (2D)→ 2D 物理耗时                                    │
│  Network    → 网络消息处理                                    │
│  UI         → Canvas/UI 重建和渲染                           │
│  Global MI  → Managed/Unmanaged 内存追踪                     │
│  Video      → 视频播放                                        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 CPU Usage 深度分析

CPU Usage 是 Unity 性能分析的核心模块：

```
Profiler CPU Usage 界面:
┌──────────────────────────────────────────────────────────────┐
│  帧时间曲线 (上方)                                            │
│  ████╗  ╔█████╗      ╔█████████████╗                        │
│       ╚══╝     ╚══════╝             ← 帧时间随时间变化       │
│                                                              │
│  [选中某一帧]                                                │
│                                                              │
│  Hierarchy 视图 (下方):                                      │
│  ┌──────────────────────────────┬────────┬────────┬────────┐│
│  │ Overview                     │ GC Alloc│ Time ms│ %      ││
│  ├──────────────────────────────┼────────┼────────┼────────┤│
│  │ PlayerLoop                   │ 0.0 B  │ 16.2   │ 100%   ││
│  │ ├─ Camera.Render             │ 0.0 B  │ 8.5    │ 52.5%  ││
│  │ │  ├─ Render.OpaqueObjects   │ 0.0 B  │ 3.2    │ 19.8%  ││
│  │ │  ├─ Render.Transparent     │ 0.0 B  │ 1.1    │ 6.8%   ││
│  │ │  ├─ PostProcessing         │ 0.0 B  │ 2.8    │ 17.3%  ││
│  │ │  └─ CullResultsVisible     │ 0.0 B  │ 0.8    │ 4.9%   ││
│  │ ├─ Update                    │ 24.0 B │ 4.2    │ 25.9%  ││
│  │ │  ├─ AIManager.Update       │ 0.0 B  │ 2.1    │ 13.0%  ││
│  │ │  ├─ Physics.Simulate       │ 0.0 B  │ 1.5    │ 9.3%   ││
│  │ │  └─ ScriptRunBehaviour     │ 24.0 B │ 0.6    │ 3.7%   ││← GC!
│  │ ├─ FixedUpdate               │ 0.0 B  │ 2.1    │ 13.0%  ││
│  │ │  └─ Physics.Simulate       │ 0.0 B  │ 2.1    │ 13.0%  ││
│  │ └─ LateUpdate               │ 0.0 B  │ 1.4    │ 8.6%   ││
│  │    └─ Animation              │ 0.0 B  │ 1.4    │ 8.6%   ││
│  └──────────────────────────────┴────────┴────────┴────────┘│
│                                                              │
│  关键列:                                                     │
│  ├─ GC Alloc: 该节点每帧的 GC 堆分配 (理想 = 0!)            │
│  ├─ Time ms:  该节点及子节点的总耗时                        │
│  ├─ %:        占总帧时间的百分比                              │
│  └─ Self:     切换视图查看自身耗时(不含子节点)              │
│                                                              │
│  → ScriptRunBehaviour 有 24B GC Alloc → 需排查              │
│  → Camera.Render 占 52.5% → 渲染是 CPU 瓶颈                 │
└──────────────────────────────────────────────────────────────┘
```

### 2.3 Timeline 视图与多线程分析

```
Timeline 视图 (多线程时间线):
时间 →  0    2    4    6    8   10   12   14   16ms
Main    ┌──────┐┌──────┐┌──────────┐┌──────┐
Thread  │Update ││Physic││  LateUpd ││Render│
        └──────┘└──────┘└──────────┘└──┬───┘
                                      │提交
Render                              ┌──┴───────────────────┐
Thread                              │  录制渲染命令         │
                                    └──────────────────────┘
Job     ┌──┐┌──┐┌──┐┌──┐┌──┐┌──┐
Workers │W1││W2││W3││W1││W2││W3│  ← Job System 任务
        └──┘└──┘└──┘└──┘└──┘└──┘

分析要点:
├─ Main Thread 空闲段 → CPU 未充分利用
├─ Render Thread 等待 Main Thread → 串行瓶颈
├─ Job Workers 空闲 → 任务分配不均
├─ Main 和 Render 并行段 → 好的并行设计
└→ 定位串行等待点, 尽量并行化
```

### 2.4 GC Alloc 追踪

GC（垃圾回收）分配是 Unity 性能的头号杀手：

```
GC Alloc 的影响:
├─ 每次 GC 分配 → 触发内存分配器
├─ 积累到阈值 → 触发 GC 回收
├─ GC 回收 → Stop-The-World 停顿 (几ms~几十ms)
├─ 停顿 → 帧尖峰 → 卡顿感
└→ 目标: 每帧 GC Alloc = 0!

常见 GC 分配来源:
┌──────────────────────────────────────────────────┐
│  foreach (容器遍历)                               │
│  → 旧版 Unity 的 foreach 会分配迭代器!           │
│  → 修复: 使用 for 循环                            │
├──────────────────────────────────────────────────┤
│  string 拼接                                     │
│  → "score: " + score → 分配新 string             │
│  → 修复: StringBuilder 或 string.Format          │
├──────────────────────────────────────────────────┤
│  LINQ                                            │
│  → list.Where().Select().ToList() → 大量分配     │
│  → 修复: 手动循环                                 │
├──────────────────────────────────────────────────┤
│  闭包捕获                                        │
│  → lambda 捕获局部变量 → 分配闭包对象             │
│  → 修复: 避免热路径中使用闭包                      │
├──────────────────────────────────────────────────┤
│  new List<T>() / new Dictionary                  │
│  → 每帧创建容器 → 分配                           │
│  → 修复: 缓存为成员变量, Clear() 复用             │
├──────────────────────────────────────────────────┤
│  Coroutine yield return new WaitForSeconds()     │
│  → 每次创建新对象                                 │
│  → 修复: 缓存 WaitForSeconds 实例                 │
├──────────────────────────────────────────────────┤
│  GetComponent<T>()                               │
│  → 部分版本有分配 (Boxing)                       │
│  → 修复: 缓存引用                                 │
├──────────────────────────────────────────────────┤
│  Debug.Log()                                     │
│  → 生成格式化字符串 → 分配                        │
│  → 修复: Release 构建使用 [Conditional] 移除     │
└──────────────────────────────────────────────────┘
```

### 2.5 Profiler 自定义标记

```csharp
// 使用 ProfilerMarker 精确测量自定义代码
using UnityEngine.Profiling;

public class EnemyAI : MonoBehaviour
{
    private static readonly ProfilerMarker s_UpdateMarker =
        new ProfilerMarker("EnemyAI.Update");
    private static readonly ProfilerMarker s_PathFindMarker =
        new ProfilerMarker("EnemyAI.PathFind");

    void Update()
    {
        s_UpdateMarker.Begin();  // 开始计时
        {
            // 逻辑更新...
            if (needNewPath)
            {
                s_PathFindMarker.Begin();
                FindPath();
                s_PathFindMarker.End();
            }
        }
        s_UpdateMarker.End();   // 结束计时
    }
}

// Profiler 中显示:
// PlayerLoop
// └─ EnemyAI.Update          2.1ms
//    └─ EnemyAI.PathFind     1.5ms  ← 子标记
//
// 优势:
// ├─ 精确到自定义逻辑段
// ├─ 可嵌套
// ├─ 零运行时开销 (仅 Profiler 连接时计时)
// └─ Release 构建自动移除
```

---

## 3. Frame Debugger

### 3.1 Frame Debugger 定位

Frame Debugger 是 Unity 内置的**逐 Draw Call 检视工具**，功能类似轻量级 RenderDoc：

```
Frame Debugger 界面:
┌──────────────────────────────────────────────────────┐
│ Window → Analysis → Frame Debugger                  │
│                                                      │
│ [Enable] ← 点击启用, 捕获当前帧                       │
│                                                      │
│ 渲染事件列表:                                        │
│ ├─ Camera.Render                                     │
│ │  ├─ Setup (Clear)                                 │
│ │  ├─ Cull                                          │
│ │  ├─ Opaque                                        │
│ │  │  ├─ Draw Mesh (Standard)  #1  ← 选中           │
│ │  │  │  Shader: Standard                           │
│ │  │  │  Pass: ForwardBase                          │
│ │  │  │  Mesh: Character (12,580 verts)            │
│ │  │  │  Material: Character_Mat                    │
│ │  │  │  Textures: _MainTex, _BumpMap              │
│ │  │  ├─ Draw Mesh (Standard)  #2                   │
│ │  │  ├─ Draw Mesh (Standard)  #3                   │
│  │  │  └─ ... (1200 个 Draw Call)                   │
│  │  ├─ Skybox                                        │
│  │  ├─ Transparent                                   │
│  │  │  └─ Draw Mesh (Standard) #1201-#1250         │
│  │  └─ Post Processing                              │
│  │     ├─ Bloom                                      │
│  │     ├─ Color Grading                              │
│  │     └─ Tonemapping                               │
│  └─ UI                                               │
│     └─ Canvas.Render                                │
│                                                      │
│ 选中 Draw Call #1 后显示:                             │
│ ├─ Shader: 使用的着色器及 Pass                        │
│ ├─ Properties: 所有材质属性                           │
│ ├─ Keywords: 启用的 Shader 关键字                     │
│ ├─ Mesh: 网格信息 (顶点数/子网格)                    │
│ ├─ Textures: 绑定的纹理列表                          │
│ └─ Render Target: 当前输出目标                        │
└──────────────────────────────────────────────────────┘
```

### 3.2 Frame Debugger 分析场景

**场景1：诊断 Draw Call 过多**

```
检查项:
├─ 总 Draw Call 数 (>2000 需优化)
├─ 同材质的 Draw Call 是否合并 (Static Batching / GPU Instancing)
├─ 是否有冗余的 Pass (如多次渲染同一物体)
├─ Shadow Pass 的 Draw Call (每个光源产生阴影 Draw Call)
├─ 后处理 Pass 数量
└─ UI Canvas 的重建频率

典型问题:
├─ 1000 个独立材质 → 无法合批
│   → 优化: 合并纹理图集 (Texture Atlas), 减少材质数
│
├─ 动态物体无法 Static Batch
│   → 优化: 使用 GPU Instancing
│
├─ 每个光源产生额外 Shadow Pass
│   → 优化: 限制光源数量, 使用烘焙光照
│
└─ UI 每帧重建
    → 优化: 分割 Canvas, 静态元素单独 Canvas
```

**场景2：诊断 SetPass Call**

```
SetPass = Shader Pass 切换次数
→ 每个 Pass 切换都有 GPU 状态切换开销
→ 目标: SetPass < Draw Call (说明有合批)
→ 危险: SetPass ≈ Draw Call (每次 Draw Call 都换 Shader)

诊断:
├─ Frame Debugger 中检查 Shader 切换模式
├─ 按材质/Shader 排序渲染顺序
├─ 合并相同 Shader 的物体到同一 Draw Call
└─ 使用 SRP Batcher (URP/HDRP)
```

---

## 4. Memory Profiler

### 4.1 Memory Profiler 定位

Unity Memory Profiler（需通过 Package Manager 安装）提供**深度内存快照分析**：

```
Memory Profiler 工作流:
1. Window → Analysis → Memory Profiler
2. 点击 "Take Snapshot" → 捕获当前内存快照
3. 在关键时间点多次快照:
   ├─ 快照1: 场景A加载后
   ├─ 快照2: 游戏运行5分钟后
   └─ 快照3: 场景A卸载后
4. 对比快照 → 检测泄漏

快照分析界面:
┌───────────────────────────────────────────────────────┐
│ Tree Map (内存占用可视化):                            │
│ ┌──────────────────────────────────────────┐         │
│ │                                          │         │
│ │  ████████████   纹理 (45%)               │         │
│ │  ██████         网格 (20%)               │         │
│ │  ████           音频 (12%)              │         │
│ │  ███            动画 (8%)               │         │
│ │  ██             脚本对象 (5%)            │         │
│ │  ░░             其他 (10%)              │         │
│ │                                          │         │
│ └──────────────────────────────────────────┘         │
│                                                       │
│ 列表视图 (按类型):                                    │
│ Type           │ Count  │ Size     │ %     │ Refs   │
│ Texture2D      │ 1,250  │ 512 MB   │ 45%   │ 2,400 │
│ Mesh           │ 850    │ 128 MB   │ 11%   │ 1,700 │
│ AudioClip      │ 120    │ 96 MB    │ 8%    │ 240   │
│ AnimationClip  │ 300    │ 64 MB    │ 6%    │ 600   │
│ Material       │ 2,000  │ 32 MB    │ 3%    │ 4,000 │
│ ...            │ ...    │ ...      │ ...   │ ...   │
│                                                       │
│ 快照对比 (A → B):                                    │
│ Type           │ Δ Count │ Δ Size  │ 状态             │
│ Texture2D      │ +45     │ +28 MB  │ ⚠ 可能泄漏       │
│ Mesh           │ -120    │ -15 MB  │ ✓ 正常释放       │
│ ParticleSystem│ +8      │ +2 MB   │ ⚠ 检查           │
└───────────────────────────────────────────────────────┘
```

### 4.2 内存引用链分析

```
泄漏对象追踪:
1. 选中疑似泄漏对象 (如 Texture2D)
2. 查看 "References" → 谁在引用它?
3. 沿引用链回溯:
   Texture2D ← Material ← Renderer ← GameObject ← Scene
   ↑ 如果 GameObject 不在场景中但仍被引用 → 泄漏

常见泄漏原因:
├─ 静态变量持有对象引用 (static List<Texture> cache)
├─ 事件订阅未取消 (event += handler, 未 -= handler)
├─ 单例未销毁 (DontDestroyOnLoad 的对象持有引用)
├─ 协程未停止 (StartCoroutine 未配对 StopCoroutine)
├─ AssetBundle 未卸载 (Unload(false) 保留资源)
├─ Resources.Load 未配对 Resources.UnloadAsset
└─ Object Pool 中对象未被回收
```

### 4.3 Unity 内存分类

```
Unity 内存三分类:
┌───────────────────────────────────────────────────────┐
│  Managed Memory (托管内存)                             │
│  ├─ C# 堆分配 (new, List, Dictionary, string)        │
│  ├─ 由 Mono/IL2CPP GC 管理                             │
│  ├─ Profiler Memory 模块的 GC Alloc 列               │
│  └─ 优化: 零 GC Alloc, 对象池化                        │
├───────────────────────────────────────────────────────┤
│  Native Memory (原生内存)                              │
│  ├─ 引擎内部 C++ 分配 (纹理/网格/音频等)              │
│  ├─ 由 Unity 管理, 不受 GC 控制                       │
│  ├─ Memory Profiler 快照的主体                        │
│  └─ 优化: 资源卸载、AssetBundle 管理                  │
├───────────────────────────────────────────────────────┤
│  Graphics Memory (GPU 内存)                            │
│  ├─ 上传到 GPU 的纹理/网格/RT                          │
│  ├─ 由图形 API 驱动管理                                │
│  ├─ Memory Profiler 显示 Graphics 列                  │
│  └─ 优化: 纹理压缩、RT 复用、降低分辨率                │
└───────────────────────────────────────────────────────┘
```

---

## 5. Profile Analyzer

### 5.1 Profile Analyzer 定位

Profile Analyzer 是 Unity 的**帧数据对比工具**，用于 A/B 测试和多帧统计分析：

```
Profile Analyzer 工作流:
1. 在 Profiler 中录制一段数据 (如 500 帧)
2. Window → Analysis → Profile Analyzer
3. 两种模式:
   ├─ Single: 分析单组 Profiler 数据的统计分布
   └─ Compare: 对比两组数据 (优化前 vs 优化后)

Compare 模式 (A/B 对比):
┌───────────────────────────────────────────────────────┐
│              │ Baseline (优化前) │ Comparison (优化后) │
│ 帧 数        │ 500 帧            │ 500 帧              │
│ 平均帧时间    │ 24.5 ms           │ 18.2 ms  ← 改善!   │
│ 中位数帧时间  │ 24.0 ms           │ 18.0 ms             │
│ P99 帧时间    │ 35.2 ms           │ 22.1 ms  ← 改善!   │
│ 最大帧时间    │ 42.1 ms           │ 25.3 ms             │
│ 帧时间方差    │ 4.8 ms            │ 2.1 ms  ← 更稳定!  │
│                                                       │
│ 按标记对比 (哪些改善了, 哪些恶化了):                   │
│ Marker           │ Baseline │ Comparison │ Δ%         │
│ Camera.Render    │ 12.3ms   │ 8.5ms      │ -31% ✓     │
│ Physics.Simulate │ 5.2ms    │ 5.1ms      │ -2%        │
│ AIManager.Update │ 4.1ms    │ 2.8ms      │ -32% ✓     │
│ PostProcessing   │ 3.5ms    │ 1.8ms      │ -49% ✓     │
│ Animation       │ 1.8ms    │ 2.1ms      │ +17% ⚠ ← 注意!│
│                                                       │
│ → Animation 恶化了 17%! 虽然总体改善, 但引入了新问题   │
│ → 需排查 Animation 变化的原因                          │
└───────────────────────────────────────────────────────┘

关键指标:
├─ P50 (中位数): 50% 帧的帧时间低于此值
├─ P99: 99% 帧低于此值 (最差 1% 的阈值)
│   → P99 比 P50 更重要! 用户感受的是最差帧
├─ 方差: 稳定性指标, 方差大 = 卡顿感强
└─ 最大值: 最差单帧 (尖峰)
```

---

## 6. Unity 特有性能瓶颈

### 6.1 渲染瓶颈

```
Unity 渲染性能关键指标 (Rendering Stats):
┌──────────────────────────────────────────────────┐
│  Draw Calls:    1,250  → 目标 <2,000           │
│  Batches:       1,250                           │
│  画 Saved by Batching: 350  → 合批了 350 个     │
│  Tris:          1.5M     → 目标 <3M (移动端)    │
│  Verts:         2.8M     → 目标 <5M             │
│  SetPass:       890      → 目标 <1,000          │
│  Shadow Casters: 320                           │
│  Render Textures: 8                            │
│  VRAM:          1.2 GB  → 移动端目标 <1.5GB    │
└──────────────────────────────────────────────────┘

常见瓶颈与优化:
├─ Draw Call 过多:
│   ├─ Static Batching (静态合批)
│   ├─ Dynamic Batching (动态合批, <300顶点)
│   ├─ GPU Instancing (实例化渲染)
│   ├─ SRP Batcher (URP/HDRP)
│   └─ 合并纹理图集减少材质数
│
├─ SetPass 过多:
│   ├─ 减少材质变体 (Shader Variant 数量)
│   ├─ 使用 SRP Batcher
│   └─ 合并相同 Shader 的 Pass
│
├─ 三角形过多:
│   ├─ LOD 系统 (细节层次)
│   ├─ Occlusion Culling (遮挡剔除)
│   ├─ Frustum Culling (视锥剔除)
│   └─ Mesh 减面
│
└─ VRAM 过大:
    ├─ 纹理压缩 (ASTC 移动端 / BCn PC)
    ├─ 纹理流式加载 (Mipmap Streaming)
    └─ RT 复用
```

### 6.2 脚本执行瓶颈

```
MonoBehaviour 生命周期开销:
┌──────────────────────────────────────────────────────┐
│  Awake() / Start()  → 一次性, 初始化                  │
│  OnEnable / OnDisable → 频繁触发, 避免重逻辑          │
│  Update()           → 每帧, 最容易成为瓶颈           │
│  FixedUpdate()      → 固定频率 (物理), 注意累积      │
│  LateUpdate()       → Update 后, 常用于相机跟随       │
│  Coroutine          → 协程, 注意分配开销              │
└──────────────────────────────────────────────────────┘

Update 性能要点:
├─ MonoBehaviour.Update 的调用开销:
│   → 每个脚本都有 Update → 10000 脚本 = 10000 次调用
│   → Native → Managed 调用有开销
│   → 优化: 使用 Manager 模式, 集中更新
│
├─ Manager 模式:
│   // 差: 10000 个 Enemy 各自 Update
│   class Enemy : MonoBehaviour {
│       void Update() { ... }  // 10000 次调用
│   }
│   
│   // 好: 单个 EnemyManager 集中更新
│   class EnemyManager : MonoBehaviour {
│       List<Enemy> enemies;
│       void Update() {
│           foreach (var e in enemies) { e.UpdateAI(); }
│           // 1 次调用, 数据局部性好
│       }
│   }
│
├─ Transform 操作开销:
│   → transform.position 每次访问有 native 调用开销
│   → 优化: 缓存 transform 引用
│   → 优化: 批量修改后一次 Apply
│
├─ GetComponent 开销:
│   → 每次调用有查找开销
│   → 优化: Start 中缓存, Update 中使用缓存引用
│
└─ Find 系列:
    → GameObject.Find / FindWithTag → O(n) 查找
    → 优化: Start 中查找并缓存, 或用事件注册替代
```

### 6.3 物理性能

```
Unity Physics 性能要点:
├─ Rigidbody 数量: 目标 <500 (3D) / <2000 (2D)
├─ Collider 复杂度: MeshCollider 远贵于 BoxCollider
├─ Collision Detection:
│   ├─ Discrete:  最快 (默认)
│   ├─ Continuous:  中等 (防穿透)
│   └─ Continuous Dynamic: 最慢 (高速物体)
├─ Fixed Timestep: 0.02 (50Hz) → 减少可提升性能但降低精度
├─ Layer Collision Matrix: 禁用不需要的层间碰撞检测
└─ Physics.autoSyncTransforms: 关闭 (手动 SyncTransforms)

优化方向:
├─ 减少动态 Rigidbody → 静态 Collider + 少量动态
├─ 用 BoxCollider/SphereCollider 替代 MeshCollider
├─ 高速物体才用 Continuous Dynamic
├─ 禁用远处物体的碰撞 (LOD for physics)
└─ 使用 Job System + Burst 并行物理 (Unity Physics)
```

---

## 7. Unity 性能分析实战工作流

```
Step 1: 开发阶段持续监控
├─ 始终开启 Profiler (Editor 模式)
├─ 关注 GC Alloc 列 (目标 0)
├─ 关注 Draw Call 数 (目标 <2000)
├─ 定期检查帧时间是否有尖峰
└─ 使用 [Conditional("UNITY_EDITOR")] 移除 Debug.Log

Step 2: 构建后基线测量
├─ 构建 Development Build (带 Profiler)
├─ 连接设备 (Android/iOS/PC)
├─ Device Profiler 远程采集
├─ 运行标准基准场景 5-10 分钟
├─ Profile Analyzer 分析 P50/P99
└→ 建立性能基线

Step 3: 瓶颈定位
├─ CPU 瓶颈?
│   ├─ Profiler CPU Usage → Hierarchy 定位热点
│   ├─ 检查 GC Alloc → 消除所有分配
│   ├─ 检查 Update 热点 → Manager 模式重构
│   └─ 检查 Camera.Render → 合批/LOD/剔除
│
├─ GPU 瓶颈?
│   ├─ Profiler GPU Usage → 各 Pass 耗时
│   ├─ Frame Debugger → Draw Call 分析
│   ├─ Rendering Stats → SetPass/三角形数
│   └→ 需要更深入: RenderDoc / NSight Graphics
│
└─ 内存瓶颈?
    ├─ Memory Profiler 快照分析
    ├─ 检查纹理/网格/音频占用
    ├─ 快照对比 → 泄漏检测
    └→ 资源管理优化

Step 4: 优化与验证
├─ 针对性优化
├─ Profile Analyzer A/B 对比
├─ 确认 P50/P99 改善
├─ 确认无新瓶颈引入
└─ 多设备验证

Step 5: 持续回归
├─ Performance Testing Framework 自动化
├─ CI/CD 集成性能回归检测
├─ 阈值告警 (帧时间 > 目标值时通知)
└→ 建立长期性能趋势监控
```

---

## 8. Unity 与原生工具联动

```
Unity Profiler 的局限 → 需要原生工具补充:

┌──────────────────────┬──────────────┬────────────────────┐
│ 分析需求              │ Unity 工具    │ 补充原生工具        │
├──────────────────────┼──────────────┼────────────────────┤
│ GPU 计数器           │ ❌ 不支持     │ NSight Graphics    │
│ Shader 指令分析      │ ❌ 不支持     │ NSight / RenderDoc│
│ 缓存命中率 (CPU)     │ ❌ 不支持     │ VTune             │
│ 系统级功耗追踪       │ ❌ 不支持     │ Snapdragon Profiler│
│ 帧捕获纹理查看       │ Frame Debugger│ RenderDoc (更深)   │
│ Draw Call 逐条检视    │ Frame Debugger│ RenderDoc / NSight│
│ 内存分配调用栈       │ Memory Profiler│ ASan (原生层)     │
│ 多线程时间线         │ Profiler Timeline│ NSight Systems  │
└──────────────────────┴──────────────┴────────────────────┘

联动策略:
├─ Unity Profiler → 定位"哪个模块慢"
├─ Frame Debugger → 定位"渲染了什么"
├─ RenderDoc/NSight → 定位"GPU 为什么慢"
└─ VTune → 定位"CPU 代码为什么慢" (原生层)

启用原生工具捕获 Unity 帧:
├─ RenderDoc: 设置环境变量或使用 "Inject into Process"
├─ NSight Graphics: 启动 Unity 进程后 Attach
├─ 需 Development Build + 多线程渲染可能影响捕获
└→ 建议在独立的测试场景中捕获, 避免编辑器干扰
```

---

## 9. 本章小结

| 工具 | 核心能力 | 适用阶段 |
|------|---------|---------|
| **Profiler (CPU Usage)** | 帧时间按模块拆分、GC Alloc 追踪 | 日常开发、瓶颈定位 |
| **Profiler (GPU Usage)** | GPU 各 Pass 耗时 | GPU 瓶颈初步定位 |
| **Frame Debugger** | 逐 Draw Call 检视、渲染状态 | 渲染问题诊断 |
| **Memory Profiler** | 内存快照、泄漏检测、引用链 | 内存分析 |
| **Profile Analyzer** | 多帧统计、A/B 对比、P99 分析 | 优化验证 |
| **Rendering Stats** | 实时 Draw Call/三角形/SetPass | 快速检查 |
| **Performance Testing** | 自动化基准测试 | CI/CD 回归 |

---

*上一篇：[04-设备与平台性能分析](04-设备与平台性能分析.md)*
*下一篇：[06-UE平台性能分析](06-UE平台性能分析.md)*
