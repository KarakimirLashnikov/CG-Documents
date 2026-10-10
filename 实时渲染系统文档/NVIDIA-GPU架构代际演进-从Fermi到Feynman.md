# NVIDIA GPU 架构代际演进：从 Fermi 到 Feynman

> **关键词**：NVIDIA GPU 架构演进、Fermi、Kepler、Maxwell、Pascal、Volta、Turing、Ampere、Ada Lovelace、Hopper、Blackwell、Rubin、Feynman、SM 演进、Tensor Core、RT Core、NVLink、HBM

> **前置阅读**：[01 - GPU 架构基础与硬件原理](01-GPU架构基础与硬件原理.md) — 理解 SM、SIMT、内存层次等基础概念

---

## 一、总览：十五年的跃进

从 2010 年 Fermi 到计划 2028 年问世的 Feynman，NVIDIA GPU 经历了令人震惊的代际跃迁。以下是核心指标的演进全景：

| 架构            | 年份   | 制程       | 晶体管数   | FP32 峰值  | 显存带宽       | 关键创新                            |
| ------------- | ---- | -------- | ------ | -------- | ---------- | ------------------------------- |
| **Fermi**     | 2010 | 40nm     | ~3B    | ~1.5 TF  | ~150 GB/s  | 首个完整计算架构、ECC                    |
| **Kepler**    | 2012 | 28nm     | ~7.1B  | ~4.3 TF  | ~288 GB/s  | SMX、Hyper-Q、Dynamic Parallelism |
| **Maxwell**   | 2014 | 28nm     | ~8B    | ~6.1 TF  | ~336 GB/s  | 能效革命、SMM 四分区                    |
| **Pascal**    | 2016 | 16nm FF  | ~15.3B | ~10.6 TF | ~732 GB/s  | NVLink 1.0、HBM2、Unified Memory  |
| **Volta**     | 2017 | 12nm FFN | ~21.1B | ~15.7 TF | ~900 GB/s  | **Tensor Core 诞生**              |
| **Turing**    | 2018 | 12nm FFN | ~18.6B | ~16.3 TF | ~616 GB/s  | **RT Core、实时光追**                |
| **Ampere**    | 2020 | 7nm/8nm  | ~54.2B | ~19.5 TF | ~2.0 TB/s  | TF32、结构化稀疏、MIG                  |
| **Ada**       | 2022 | TSMC 4N  | ~76.3B | ~82.6 TF | ~1.0 TB/s  | SER、DLSS 3 FrameGen             |
| **Hopper**    | 2022 | TSMC 4N  | ~80B   | ~67 TF   | ~3.35 TB/s | Transformer Engine、TMA          |
| **Blackwell** | 2024 | TSMC 4NP | ~208B  | —        | ~8 TB/s    | 第5代 TC (FP4)、Dual-Die           |
| **Rubin**     | 2026 | TSMC 3nm | ~336B  | —        | ~22 TB/s   | HBM4、第6代 TC (NVFP4)             |
| **Feynman**   | 2028 | 待定       | 待定     | 待定       | 待定         | 下一代 HBM                         |

> **注意**：Ampere (2020)、Ada Lovelace (2022)、Hopper (2022) 是 Turing 到 Blackwell 之间的重要过渡架构，将分别在下方详细介绍。

---

## 二、架构演进的五大主线

在深入每一代之前，先识别驱动演进的五条主线：

```
                     ┌─────────────────────────────────────────┐
                     │       NVIDIA GPU 架构演进五大主线         │
                     └─────────────────────────────────────────┘
                                      │
          ┌───────────────┬───────────┼───────────┬───────────────┐
          │               │           │           │               │
    ① SM 计算单元     ② 专用加速器  ③ 内存与互联  ④ 制造工艺     ⑤ 软件生态
          │               │           │           │               │
    Fermi: 32 CUDA    Volta: TC   Kepler:GPU Boost  40nm→3nm    CUDA
    Kepler: 192 CUDA  Turing: RT  Pascal:NVLink   →A16?        TensorRT
    Maxwell: 128分4区  Ampere:     Volta: HBM2                  cuDNN
    Pascal: 64→128    稀疏+TF32   Blackwell:                    DLSS
    Volta: 独立线程     Hopper:     NVLink 5                     Transformer Engine
    调度              Transformer Rubin: HBM4
         ...           Engine       + NVLink 6
                       Blackwell:   Feynman: 下一代 HBM
                       FP4
```

---

## 三、逐代详解

### 3.1 Fermi (2010) — 通用 GPU 计算的黎明

> **代表产品**：GTX 480 (GF100)、GTX 580 (GF110)、Tesla M2090
> **代号含义**：恩里科·费米 (Enrico Fermi)，意大利物理学家，核能先驱

**历史地位**：Fermi 是 NVIDIA 第一个"从设计之初就为通用计算服务"的 GPU 架构。它完成了 GPU 从"纯粹图形加速器"到"通用并行处理器"的质变。

**SM 架构**：

```
┌──────────────────────────────────────────────┐
│               Fermi SM（16 个）                │
│  ┌──────────────────────────────────────┐    │
│  │   32 CUDA Cores（排成 2 组 × 16）     │    │
│  └──────────────────────────────────────┘    │
│  ┌──────────────────────────────────────┐    │
│  │   16 Load/Store 单元                  │    │
│  └──────────────────────────────────────┘    │
│  ┌──────────────────────────────────────┐    │
│  │   4 个特殊功能单元（SFU）— sin/cos/sqrt  │   │
│  └──────────────────────────────────────┘    │
│  ┌────────────────────────────────────────┐  │
│  │   64 KB 可配置 L1/Shared Memory        │  │
│  │   （可选 16 KB SM + 48 KB L1 / 48+16）  │  │
│  └────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────┐  │
│  │   2 个 Warp Scheduler + 2 个 Dispatch   │  │
│  └────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘
```

**关键突破**：

| 特性 | 意义 |
|------|------|
| **完整 IEEE 754-2008 FP64** | 首次支持双精度浮点标准，使 GPU 被科学计算社区接纳 |
| **ECC 内存（Tesla 系列）** | 检测并纠正内存错误，满足 HPC 可靠性要求 |
| **L1/L2 缓存层次** | 引入传统的 CPU 式缓存体系，减少显存访问延迟 |
| **C++ 原生支持** | GPU 端代码不再局限于类 C 的着色器语言 |
| **Unified Addressing** | CPU/GPU 共享同一 64 位地址空间，简化编程模型 |

**算力规格**（GTX 580 / GF110）：

| 指标 | 数值 |
|------|------|
| SM 数量 | 16 |
| CUDA Cores | 512 |
| 核心频率 | 772 MHz |
| FP32 单精度 | 1.58 TFLOPS |
| FP64 双精度 | 0.20 TFLOPS (1/8 rate) |
| 显存 | 1.5 GB GDDR5，192.4 GB/s |
| TDP | 244 W |
| 晶体管 / 面积 | 3.0B / 520 mm² |

> **影响**：Fermi 奠定了 CUDA 生态的基础。没有 Fermi 将 ECC、C++、完整 FP64 引入 GPU，HPC 社区就不会认真考虑 GPU 加速。

---

### 3.2 Kepler (2012) — 性能与可编程性的双重飞跃

> **代表产品**：GTX 680 (GK104)、GTX Titan (GK110)、Tesla K40/K80
> **代号含义**：约翰内斯·开普勒 (Johannes Kepler)，德国天文学家，行星运动定律

**历史地位**：Kepler 大幅改进了 SM 设计——从 32 CUDA Core/SM 跃升到 192 CUDA Core/SM（SMX），并引入了多项突破性的软件能力。

**SM 架构——SMX 革命**：

```
┌────────────────────────────────────────────────────────────┐
│                    Kepler SMX（GK110：15 个）                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │    192 CUDA Cores（单精度） + 64 DP Units（双精度）    │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │    32 特殊功能单元（SFU）                              │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │    4 个 Warp Scheduler + 8 个 Dispatch Unit            │  │
│  │    每个 scheduler 可双发射（同时发 2 条独立指令）        │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │    64 KB 可配置 L1/Shared Memory + 48 KB Read-Only    │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │    1.5 MB L2 Cache（全芯片共享）                        │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

**关键突破**：

| 特性 | 意义 |
|------|------|
| **SMX 设计** | 单 SM 容纳 192 CUDA Cores，总数高达 2880，吞吐量飞跃 |
| **Hyper-Q** | CPU 端 32 个并行工作队列同时向 GPU 推送任务，消除假依赖 |
| **Dynamic Parallelism** | GPU kernel 可以在不返回 CPU 的情况下启动新的 kernel |
| **GPUDirect RDMA** | GPU 间/网卡间直接数据传输，绕过 CPU |
| **GPU Boost** | 动态频率调节，在功耗/温度允许范围内自动超频 |
| **Read-Only Data Cache** | 48 KB 独立只读纹理缓存，提升带宽利用效率 |

**算力规格**（Tesla K40 / GK110）：

| 指标 | 数值 |
|------|------|
| SMX 数量 | 15 |
| CUDA Cores | 2880 |
| FP32 | 4.29 TFLOPS |
| FP64 | 1.43 TFLOPS (1/3 rate) |
| 显存 | 12 GB GDDR5，288 GB/s |
| TDP | 235 W |
| 晶体管 / 面积 | 7.1B / 551 mm² |

> **影响**：Kepler 的 Hyper-Q 和 Dynamic Parallelism 解决了 GPU 多任务调度和嵌套并行的核心痛点，让 GPU 不再只是"一卡跑一核"的简单加速器。K80 双 GPU 卡成为了第一代 AI 深度学习浪潮的主力计算卡。

---

### 3.3 Maxwell (2014) — 能效革命

> **代表产品**：GTX 750 Ti (GM107)、GTX 980 (GM204)、GTX Titan X (GM200)
> **代号含义**：詹姆斯·克拉克·麦克斯韦 (James Clerk Maxwell)，苏格兰物理学家，电磁理论

**历史地位**：Maxwell 是一次"不做加法做减法"的架构革命，通过重新设计 SM 微架构，在相同 28nm 制程下实现了接近 2× 的能效提升。

**SM 架构——SMM（Maxwell SM）**：

```
┌────────────────────────────────────────────────────────────┐
│                  Maxwell SMM（GM200：24 个）                   │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              SMM 被分为 4 个独立子分区：               │  │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐    │  │
│  │  │ 32 CUDA │ │ 32 CUDA │ │ 32 CUDA │ │ 32 CUDA │    │  │
│  │  │ 1 Warp  │ │ 1 Warp  │ │ 1 Warp  │ │ 1 Warp  │    │  │
│  │  │ Sched.  │ │ Sched.  │ │ Sched.  │ │ Sched.  │    │  │
│  │  │ 8 SFU   │ │ 8 SFU   │ │ 8 SFU   │ │ 8 SFU   │    │  │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘    │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │    96 KB 专用 Shared Memory（不再与 L1 共享）          │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │    2 MB L2 Cache（全芯片共享）                         │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

**分区设计的精妙之处**：

Kepler SMX 中 192 个 CUDA Core 由 4 个 Warp Scheduler 共享调度，控制逻辑和寄存器文件的扇出/扇入开销极大。Maxwell 将 SMM 划分为 4 个独立子分区，每个子分区有自己的 32 CUDA Core + 1 Warp Scheduler + 部分寄存器文件：

- **降低了控制开销**：每个 Warp Scheduler 只管理 32 个核，而非 48 个
- **寄存器访问更高效**：局部化的寄存器文件减少了线延迟
- **功耗大幅降低**：闲置的子分区可直接时钟门控 (Clock Gating)

**关键突破**：

| 特性 | 意义 |
|------|------|
| **能效跃升** | 同制程下，每瓦性能达到 Kepler 的 ~2× |
| **SMM 四分区** | 模块化 SM 设计，降低控制与供电开销 |
| **Dedicated Shared Memory** | 96 KB 独立 SMem，不再与 L1 争抢 |
| **Tiled Caching** | 2D 平铺缓存策略提升图形渲染命中率 |
| **MFAA** | 多帧采样抗锯齿，同画质更低开销 |
| **VXGI** | 体素锥追踪全局光照（消费级 GI 探索） |

**能效对比**：

| | Kepler GTX 680 (GK104) | Maxwell GTX 980 (GM204) |
|--|------------------------|------------------------|
| CUDA Cores | 1536 | 2048 |
| TDP | 195 W | 165 W |
| FP32 | 3.09 TFLOPS | 4.61 TFLOPS |
| **每瓦 FP32** | **15.8 GF/W** | **27.9 GF/W** |
| 晶体管 | 3.54B | 5.2B |
| 面积 | 294 mm² | 398 mm² |

> **影响**：Maxwell 证明了"架构效率"不亚于"制程升级"的价值。它的 SMM 分区设计思想深刻影响了后续 Pascal、Turing 的 SM 微架构。GTX 970/980 也是许多游戏玩家和初代深度学习研究者的入门卡。

---

### 3.4 Pascal (2016) — NVLink 与 HBM 时代开启

> **代表产品**：Tesla P100 (GP100)、GTX 1080 Ti (GP102)、Titan Xp、TITAN V (GV100)
> **代号含义**：布莱兹·帕斯卡 (Blaise Pascal)，法国数学家、物理学家，流体力学与概率论先驱

**历史地位**：Pascal 是"两大条线"的分水岭——数据中心用 GP100（HBM2 + NVLink），消费级用 GP102/GP104（GDDR5X）。此外，16nm FinFET 是制程上的重要一步。

**双线策略**：

```
Pascal 架构
├── GP100（数据中心）
│   ├── 60 SM × 64 CUDA Cores = 3840 CUDA Cores
│   ├── HBM2 16 GB，732 GB/s
│   ├── NVLink 1.0 (160 GB/s GPU-GPU)
│   ├── FP64 达 FP32 的 1/2 rate（科学计算友好）
│   └── 15.3B 晶体管 / 610 mm² / TSMC 16nm FinFET
│
└── GP102（消费级旗舰）
    ├── 30 SM × 128 CUDA Cores = 3840 CUDA Cores
    ├── GDDR5X 11 GB，484 GB/s（1080 Ti）
    ├── 无 NVLink，无 HBM2
    ├── FP64 降级（1/32 rate，成本控制）
    └── 12B 晶体管 / 471 mm²
```

> **注意**：GP100 的 SM 设计（64 CUDA Core/SM）是"低密度 SM"，侧重 FP64 和 HPC。GP102 的 SM（128 CUDA Core/SM）是"高密度 SM"，侧重图形和游戏。

**关键突破**：

| 特性 | 意义 |
|------|------|
| **NVLink 1.0** | GPU 间直连，单方向 160 GB/s，PCIe 3.0 x16 的 ~10× |
| **HBM2（GP100）** | 3D 堆叠内存，高带宽低功耗，单 P100 即 732 GB/s |
| **16nm FinFET** | 相比 Maxwell 28nm 平面工艺，密度和功耗双重飞跃 |
| **Unified Memory 增强** | Page Migration Engine：自动迁移 CPU/GPU 页，无需显式 cudaMemcpy |
| **Compute Preemption** | kernel 执行可被抢占暂停，支持指令级上下文切换 |
| **FP16 双速（GP100）** | FP16 运行在 FP32 的 2× 速率（21.2 TFLOPS） |

**NVLink 拓扑**：

```
      CPU                          CPU
       │                            │
  ┌────┴────┐                  ┌────┴────┐
  │ PCIe    │                  │ PCIe    │
  │ Switch  │                  │ Switch  │
  └────┬────┘                  └────┬────┘
       │                            │
   ┌───┴───┐                    ┌───┴───┐
   │ GPU 0 │ ═══ NVLink ═══    │ GPU 2 │
   └───┬───┘    160 GB/s   ┌───┴───┐   │
       │                   │ GPU 3 │───┘
   ┌───┴───┐               └───┬───┘
   │ GPU 1 │ ═══ NVLink ═══    │
   └───────┘                   │
      ┌────────────────────────┘
      └── 4 GPU 可以有全互联或线型拓扑
```

**算力规格**：

| | Tesla P100 (GP100) | GTX 1080 Ti (GP102) |
|--|---------------------|----------------------|
| SM 数量 | 56 (启用) | 28 |
| CUDA Cores | 3584 | 3584 |
| FP32 | 9.3 TFLOPS | 11.3 TFLOPS |
| FP64 | 4.7 TFLOPS | 0.35 TFLOPS |
| 显存 | 16 GB HBM2 | 11 GB GDDR5X |
| 显存带宽 | 720 GB/s | 484 GB/s |
| TDP | 250 W | 250 W |
| NVLink | 160 GB/s | 无 |

> **影响**：Pascal 是深度学习爆发期的"主力"。P100 的 NVLink 在多卡训练中首次突破了 PCIe 瓶颈；GTX 1080 Ti 成为有史以来最受欢迎的消费级 GPU 之一，至今仍有大量遗留部署。

---

### 3.5 Volta (2017) — AI 加速的奇点：Tensor Core 诞生

> **代表产品**：Tesla V100 (GV100)
> **代号含义**：亚历山德罗·伏特 (Alessandro Volta)，意大利物理学家，电池发明者

**历史地位**：Volta 是 GPU 历史上最重要的转折点之一。**Tensor Core 的引入**标志着 GPU 从"并行矢量化处理器"进化为"矩阵乘法专用加速器"，直接开启了深度学习的大规模训练时代。

```
┌─────────────────────────────────────────────────────────────────┐
│                     Volta 的里程碑意义                            │
│                                                                 │
│   在 Volta 之前，GPU = 图形加速 + 通用并行（CUDA Cores）          │
│   在 Volta 之后，GPU = 图形加速 + 通用并行 + AI 专用（Tensor Core）│
│                                                                 │
│   这不是性能堆砌，这是功能类别的扩展                               │
└─────────────────────────────────────────────────────────────────┘
```

**V100 SM 架构**：

```
┌──────────────────────────────────────────────────────────────┐
│                    Volta SM（V100：80 个）                       │
│  ┌────────────────────────────────────────────────────────┐  │
│  │            4 个处理块（Sub-core）                        │  │
│  │  ┌───────────────┐  ┌───────────────┐                  │  │
│  │  │ 16 FP32 Cores │  │ 16 FP32 Cores │  ... 共 64 FP32 │  │
│  │  │ 8 FP64  Cores │  │ 8 FP64  Cores │  ... 共 32 FP64 │  │
│  │  │ 8 LD/ST Units │  │ 8 LD/ST Units │                  │  │
│  │  │ 2 TENSOR CORE │  │ 2 TENSOR CORE │  !! 关键新增 !!  │  │
│  │  │ 1 Warp Sched. │  │ 1 Warp Sched. │   共 8 Tensor    │  │
│  │  │ 1 Dispatch    │  │ 1 Dispatch    │   Cores / SM     │  │
│  │  └───────────────┘  └───────────────┘                  │  │
│  └────────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  128 KB L1 / Shared Memory（可配置：最多 96 KB SMem）    │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

**Tensor Core 原理概览**：

```
              FP16 × FP16 + FP32 融合乘加

     A (4×4 FP16)       B (4×4 FP16)       C/D (4×4 FP32)
    ┌           ┐      ┌           ┐       ┌           ┐
    │ a  a  a  a│      │ b  b  b  b│       │ c' c' c' c│
    │ a  a  a  a│  ×   │ b  b  b  b│  + C =│ c' c' c' c│
    │ a  a  a  a│      │ b  b  b  b│       │ c' c' c' c│
    │ a  a  a  a│      │ b  b  b  b│       │ c' c' c' c│
    └           ┘      └           ┘       └           ┘

    一 cycle 完成 4×4×4 = 64 FMA = 128 FP ops
    一个 V100 Tensor Core → 每 cycle 128 FP16 FMA
    640 个 Tensor Core × 1.455 GHz → 125 TFLOPS FP16!
```

**关键突破**：

| 特性 | 意义 |
|------|------|
| **Tensor Core V1** | 硬件原生 4×4 矩阵乘加，AI 训练加速 3–5×（相比 P100） |
| **独立线程调度** | 每个线程有独立 PC 和栈，同一 Warp 内线程可交错执行（不再是严格的锁步） |
| **NVLink 2.0** | 单方向 300 GB/s（P100 的 2×），支持 6 GPU 全互联 |
| **HBM2 32 GB** | 当时最大单卡显存，使 GPT-2 级别模型可单卡训练 |
| **6 NVLINK 通道** | V100 支持构建 NVSwitch 架构的 DGX-1 |

**算力规格**（V100 SXM2 32 GB）：

| 指标 | 数值 |
|------|------|
| SM 数量 | 80 |
| CUDA Cores | 5120 |
| Tensor Cores | 640 |
| FP32 | 15.7 TFLOPS |
| FP64 | 7.8 TFLOPS |
| FP16 Tensor Core | 125 TFLOPS |
| 显存 | 32 GB HBM2，900 GB/s |
| NVLink | 300 GB/s × 6 通道 |
| TDP | 300 W |
| 晶体管 / 面积 | 21.1B / 815 mm² |

> **影响**：V100 是 AI 历史上最重要的计算硬件之一。它在 2017-2020 年间是深度学习训练的事实标准。Transformer 论文和 GPT-2/BERT 等里程碑工作都是在 V100 集群上完成的。有了 V100，NVIDIA 的市值开始以指数级加速增长。

---

### 3.6 Turing (2018) — 实时光线追踪的降临

> **代表产品**：RTX 2080 Ti (TU102)、RTX 2070、Titan RTX
> **代号含义**：艾伦·图灵 (Alan Turing)，英国数学家、计算机科学之父

**历史地位**：Turing 首次将 **实时光线追踪硬件（RT Core）** 带入消费级 GPU，同时将 Tensor Core 引入消费级产品线（用于 DLSS）。这是图形学从"光栅化模拟"到"物理正确渲染"的分水岭。

**TU102 SM 架构**：

```
┌──────────────────────────────────────────────────────────────┐
│                  Turing SM（TU102：72 个）                      │
│  ┌────────────────────────────────────────────────────────┐  │
│  │          4 个处理块（Sub-core）                          │  │
│  │  ┌───────────────┐  ┌───────── ...（子块详情）──┐        │  │
│  │  │ 16 INT32 Core │  │                           │        │  │
│  │  │ 16 FP32 Core  │  │  Turing 的关键创新：        │        │  │
│  │  │ 2 TENSOR CORE │  │  INT32 与 FP32 可并发执行！ │        │  │
│  │  │ 8 LD/ST Units │  │  不像前代，整数和浮点抢发射  │        │  │
│  │  │ 1 RT CORE ⚡   │  │                            │        │  │
│  │  │ 1 Warp Sched. │  │                            │        │  │
│  │  │ 1 Dispatch    │  │                            │        │  │
│  │  └───────────────┘  └────────────────────────────┘        │  │
│  └────────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  96 KB L1 / Shared Memory（可配置：64 KB SMem + 32 L1）│  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

**RT Core 工作流**：

```
传统光栅化：三角形 → 顶点着色 → 光栅化 → 片元着色 → 输出
    无反射/折射/间接阴影，全靠 hack（cubemap、SSR 等）

光追管线：   像素 → Ray Generation（GPU）
                    ↓
              BVH 遍历 → Ray-Triangle 求交
                ↑               ↓
                │         ┌─────────────┐
                │  CPU    │  RT Core 🔥  │ ← 硬件固定函数
                │  或      └─────────────┘     专做射线-BVH/三角求交
                │  GPU               ↓
                └───── 遍历结果 → Shading → 如命中新材质 → 再射 Ray
```

RT Core 硬件化两个最耗时的操作：
1. **BVH 遍历**：在包围盒层次结构中查找射线与哪些包围盒相交
2. **Ray-Triangle 求交**：精确计算射线是否命中三角形、命中距离和重心坐标

**关键突破**：

| 特性 | 意义 |
|------|------|
| **RT Core** | 硬件光线追踪求交，相比纯 Compute Shader 加速 ~10× |
| **Tensor Core V2（消费级）** | INT8/INT4 支持，DLSS 实时超分辨率 |
| **并发 INT32 + FP32** | 同一 SM 内整数和浮点可同时执行，提升指令级并行度 |
| **Variable Rate Shading** | 画面不同区域以不同着色率渲染，降低无效计算 |
| **Mesh Shader** | 新的可编程几何管线，替代传统顶点/几何着色器 |

**算力规格**（RTX 2080 Ti / TU102）：

| 指标 | 数值 |
|------|------|
| SM 数量 | 68 |
| CUDA Cores | 4352 |
| RT Cores | 68 |
| Tensor Cores | 544 |
| FP32 | 13.4 TFLOPS |
| FP16 Tensor Core | 107.6 TFLOPS |
| 显存 | 11 GB GDDR6，616 GB/s |
| TDP | 260 W |
| 晶体管 / 面积 | 18.6B / 754 mm² |

> **影响**：Turing 确立了"每个 SM 都装 Tensor Core"的设计哲学（从数据中心到消费级全线受益）。RT Core 开启了实时光线追踪时代，迫使竞争对手跟进。DLSS 推动了 AI 超分辨率在消费级的普及。但 Turing 的定价策略（RTX 2080 Ti 首发 $1199）也引发了关于 GPU 价格膨胀的长期争议。

---

### 3.7 Ampere (2020) → Ada Lovelace (2022) → Hopper (2022) — 三路齐发

> Turing 之后，NVIDIA 形成了消费级（Ampere → Ada）+ 数据中心（Ampere A100 → Hopper）的双线战略，且节奏交错。此处将三代合并讲解，因为它们是 Blackwell 之前承上启下的关键过渡。

#### Ampere (2020) — 两线并行

| | **A100（数据中心）** | **RTX 3090（消费级）** |
|--|-----------------------|-------------------------|
| 代号 | GA100 | GA102 |
| 制程 | TSMC 7nm | Samsung 8nm |
| 晶体管 | 54.2B | 28.3B |
| SM 数量 | 108 | 82 |
| CUDA Cores | 6912 | 10496 |
| Tensor Core | **第 3 代**（BF16、TF32） | 第 3 代 |
| FP32 | 19.5 TFLOPS | 35.6 TFLOPS |
| FP16 TC | 312 TFLOPS | 142 TFLOPS |
| 显存 | 40/80 GB HBM2e | 24 GB GDDR6X |
| 显存带宽 | 1.6/2.0 TB/s | 936 GB/s |
| NVLink | 3.0 (600 GB/s) | 无（SLI 废弃） |
| TDP | 400 W | 350 W |

**Ampere 关键创新**：

| 创新 | 说明 |
|------|------|
| **TF32** | "Tensor Float 32"——19 位尾数 + 8 位指数 + 1 符号 = 19 bit 内部精度。AI 训练时无需改代码，替换 FP32 即获 ~2× 加速 |
| **BF16** | Brain Float 16（Google 提出），与 FP32 尾数相同但指数位少，数值范围一致——训练更稳定 |
| **结构化稀疏** | 权重矩阵 2:4 稀疏模式，Tensor Core 可双倍吞吐（理论 2× 算力） |
| **MIG** | Multi-Instance GPU——A100 可被划分为最多 7 个独立 GPU 实例，互相隔离——为云 GPU 多租户而生 |
| **Async Copy** | `memcpy_async`：数据直接从 Global Memory 异步拷贝到 Shared Memory，旁路 L1 和寄存器 |

#### Ada Lovelace (2022) — 消费级性能怪兽

| 指标 | RTX 4090 (AD102) |
|------|-----------------|
| 制程 | TSMC 4N (定制 5nm) |
| 晶体管 | 76.3B |
| SM 数量 | 128 |
| CUDA Cores | 16384 |
| RT Cores | 第 3 代（2× Turing 光追能力） |
| Tensor Cores | **第 4 代**（FP8 支持） |
| FP32 | 82.6 TFLOPS |
| 显存 | 24 GB GDDR6X，1.0 TB/s |
| TDP | 450 W |

**Ada 关键创新**：

| 创新 | 说明 |
|------|------|
| **Shader Execution Reorder (SER)** | 动态重组 Warp 中的线程执行顺序，减少光追 Shader 中的线程发散——光追性能可提升 ~2× |
| **DLSS 3 Frame Generation** | 光流加速器硬件生成插值帧（不是重复帧），理论上 FPS 可翻倍 |
| **第 3 代 RT Core** | 新增 Displaced Micro-Mesh 引擎和 Opacity Micro-Map——复杂几何体光追性能提升 |
| **FP8 Tensor Core** | 浮点 8 位训练与推理，相比 FP16 算力再翻倍 |

#### Hopper (2022) — 数据中心的 AI 引擎

| 指标 | H100 SXM5 |
|------|-----------|
| 制程 | TSMC 4N |
| 晶体管 | 80B |
| SM 数量 | 132 |
| CUDA Cores | 16896 |
| Tensor Cores | 第 4 代 (528) |
| FP32 | 67 TFLOPS |
| FP64 | 34 TFLOPS (1/2 rate) |
| FP16 TC | 989 TFLOPS |
| FP8 TC | 1979 TFLOPS |
| 显存 | 80 GB HBM3，3.35 TB/s |
| NVLink | 4.0 (900 GB/s) |
| TDP | 700 W |

**Hopper 关键创新**：

| 创新 | 说明 |
|------|------|
| **Transformer Engine** | 自动在 FP8/FP16 之间动态分配精度。Transformer 每层只需要计算一次统计量，在数值安全和吞吐间平衡——训练吞吐量提升 9× |
| **TMA（Tensor Memory Accelerator）** | 硬件异步拷贝 + 地址计算引擎。将数据搬运和地址计算从 CUDA Core 卸载到专用硬件 |
| **DPX 指令** | 动态规划专用指令，用于基因组、路径优化等算法加速 ~7× |
| **NVLink 4.0 + NVSwitch 3** | 单 GPU 900 GB/s，构建 256 GPU 的 DGX SuperPOD |
| **HBM3 首次部署** | 相比 HBM2e，带宽提升 ~1.5×，容量从 80 GB 起步 |

---

### 3.8 Blackwell (2024) — Dual-Die 与 FP4 时代的巨型引擎

> **代表产品**：B100、B200、GB200 NVL72
> **代号含义**：大卫·布莱克韦尔 (David Blackwell)，美国统计学家、博弈论先驱

**历史地位**：Blackwell 是第一个采用 **双芯片封装（Dual-Die）** 的数据中心 GPU 架构，单卡集成 2080 亿晶体管。它标志着单芯片 Scaling 遇到制程瓶颈后，NVIDIA 选择"粘合多个大 Die"的路线。

```
┌──────────────────────────────────────────────────────────────┐
│                    Blackwell 系统级架构                         │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │               GB200 Grace Blackwell Superchip         │    │
│  │  ┌──────────┐    NVLink-C2C     ┌──────────────────┐ │    │
│  │  │  Grace   │◄══════════════════►│                  │ │    │
│  │  │  CPU     │    1.8 TB/s      │  Blackwell GPU   │ │    │
│  │  │ (72核ARM)│                   │  ┌──────┬──────┐ │ │    │
│  │  └──────────┘                   │  │ Die 0│ Die 1│ │ │    │
│  │                                 │  │104B  │104B  │ │ │    │
│  │                                 │  │ Tran │ Tran │ │ │    │
│  │                                 │  │      │      │ │ │    │
│  │                                 │  │ ~10 TB/s Die │ │ │    │
│  │                                 │  │   Intercon.  │ │ │    │
│  │                                 │  └──────┴──────┘ │ │    │
│  │                                 │  192 GB HBM3e   │ │    │
│  │                                 │  8 TB/s Bandwidth│ │    │
│  │                                 └──────────────────┘ │    │
│  └──────────────────────────────────────────────────────┘    │
│                           │                                  │
│                    NVLink 5.0                                │
│                1.8 TB/s per GPU                              │
│                           │                                  │
│              ┌────────────┴────────────┐                     │
│              │    NVSwitch (第4代)      │                     │
│              │  72 GPU 全互联 @ 130 TB/s│                    │
│              │  GB200 NVL72 机架        │                     │
│              └─────────────────────────┘                     │
└──────────────────────────────────────────────────────────────┘
```

**Blackwell Tensor Core — 第5代**：

```
精度支持演变：
  P100:                FP16
  V100:                FP16
  A100:         BF16 + TF32
  H100:  FP8                 (第4代)
  B100:  FP4 + FP6 + FP8     (第5代)  ← FP4首次工程可用！

FP4 的代价/收益：
  精度：~2.5 位有效精度（vs FP8 的 3-4 位）
  带宽节省：2×（vs FP8）
  算力提升：2×（每 Tensor Core）
  适用场景：低精度推理、KV Cache 压缩
```

**关键创新**：

| 创新 | 说明 |
|------|------|
| **Dual-Die 封装** | 两颗各 104B 晶体管芯片通过 10 TB/s 片间互联，逻辑上是一个 GPU |
| **第5代 Tensor Core (FP4)** | 原生 FP4/FP6 精度，推理能效大幅提升 |
| **第2代 Transformer Engine** | 微张量缩放 + 动态精度切换，实际负载利用率更高 |
| **RAS Engine** | 可靠性/可用性/可维护性专用引擎，100% 自查自纠 |
| **Decompression Engine（解码引擎）** | 硬件数据解压引擎，有效内存容量提升 ~4× |
| **NVLink 5.0** | 单 GPU 1.8 TB/s，配合 NVSwitch 4 构建 72 GPU 机架 |

**算力规格**（B200）：

| 指标 | 数值 |
|------|------|
| 晶体管 | 208B (2×104B) |
| 显存 | 192 GB HBM3e，8 TB/s |
| FP4 (Tensor Core) | ~20 PFLOPS |
| FP8 (Tensor Core) | ~10 PFLOPS |
| FP64 | ~90 TFLOPS |
| NVLink | 1.8 TB/s (per GPU) |
| Die-Die BW | ~10 TB/s |
| TDP | ~1000 W |

> **影响**：Blackwell 是 NVIDIA 市值冲向 3 万亿的核心硬件。GB200 NVL72 将 72 GPU 封装为一个 130 TB/s 带宽的统一内存域，使万亿参数 MoE 模型可在单个机架内完成推理——这改变了 AI 基础设施的设计范式。

---

### 3.9 Rubin (2026/2027) — HBM4 与 3nm 时代的算力巨人

> **代号含义**：薇拉·鲁宾 (Vera Rubin)，美国天文学家，暗物质研究先驱

**历史地位**：Rubin 是 NVIDIA 第一个采用 TSMC 3nm 制程的 GPU 架构，晶体管数达到惊人的 3360 亿，HBM4 首次部署。它是 Blackwell 的全面进化版，目标是在相同功耗下提供 10× 推理性能。

**完整平台——Vera Rubin 的七大芯片**：

```
┌──────────────────────────────────────────────────────────────┐
│                 Vera Rubin 平台（七大芯片）                     │
│                                                              │
│  1. Vera CPU       ← 替代 Grace，88 核定制 ARMv9.2           │
│  2. Rubin GPU      ← 336B 晶体管，288 GB HBM4                │
│  3. NVLink 6 Switch ← 3.6 TB/s per GPU，NVL72 聚合 260 TB/s  │
│  4. ConnectX-9     ← 下一代 SuperNIC                         │
│  5. BlueField-4    ← DPU，数据处理卸载                        │
│  6. Spectrum-6     ← 以太网交换机，CPO 光子互联                │
│  7. BlueField-4 STX ← 存储处理器                              │
└──────────────────────────────────────────────────────────────┘
```

**Rubin 与 Blackwell 核心对比**：

| | Blackwell B200 | Rubin R200 |
|--|---------------|------------|
| 晶体管 | 208B | **336B** (+62%) |
| 制程 | TSMC 4NP | **TSMC 3nm (N3P)** |
| SM 数量 | — | 224 |
| Tensor Core | 第 5 代 (FP4) | **第 6 代 (NVFP4)** |
| Transformer Engine | 第 2 代 | **第 3 代（自适应压缩）** |
| 显存 | 192 GB HBM3e | **288 GB HBM4** |
| 显存带宽 | 8 TB/s | **22 TB/s** (+175%) |
| FP4 推理 | ~20 PFLOPS | **50 PFLOPS** |
| NVLink | 5.0, 1.8 TB/s | **6.0, 3.6 TB/s** |
| TDP | ~1000 W | **1800–2300 W** (100% 液冷) |

**HBM4 详解**：

```
HBM4 vs HBM3e：
  ┌──────────────┬─────────┬──────────┐
  │    规格       │ HBM3e   │  HBM4    │
  ├──────────────┼─────────┼──────────┤
  │ 单堆栈容量    │ 24 GB   │ 36 GB    │
  │ 接口宽度      │ 1024 bit│ 2048 bit │
  │ 每引脚速率    │ ~8 Gbps │ >11 Gbps │
  │ 单堆栈带宽    │ ~1.2 TB │ >3.0 TB  │
  │ Rubin 总堆栈  │ —       │ 8 堆栈   │
  └──────────────┴─────────┴──────────┘
```

> **影响**：Rubin 的 100% 液冷和 2300W TDP 意味着 GPU 节点正在向"整机架为基本单元"演进。NVIDIA 不再卖单卡，而是卖 NVL72 机架。推理成本降低到 Blackwell 的 1/10——这是通向 AI 大规模商业化必需的硬件基础。

---

### 3.10 Feynman (2028) — 通往 AI 推理时代的下一个里程碑

> **代号含义**：理查德·费曼 (Richard Feynman)，美国理论物理学家，量子电动力学之父、纳米技术概念提出者

**目前已知信息**（截至 GTC 2025 公告）：

| 维度 | 信息 |
|------|------|
| 发布时间 | 计划 2028 年（预期下半年的 GTC 发布） |
| 配套 CPU | Vera（与 Rubin 系列共用） |
| 内存 | **下一代 HBM**（HBM4e 或 HBM5） |
| 网络 | NVSwitch 第 8 代 (NVL-Next)，Spectrum-7 (CPO 光子)，ConnectX-10 |
| 制程 | 推测 TSMC A16 (1.6nm) 或 N2P |

**"Feynman"命名的深意**：

费曼是量子计算的先驱之一。这个名字的选择或许暗示 NVIDIA 正在为后摩尔时代的计算范式（量子-经典混合？光学互联？存内计算？）铺路。

**现阶段的关键信号**：

- **Groq LPU 技术许可**：2026 年，NVIDIA 斥资 200 亿美元获取 Groq 的 LPU (Language Processing Unit) 核心技术许可。Groq 的推理延迟极低（单 token 微秒级），可能影响 Feynman 的推理加速设计
- **X3D 堆叠传闻**：三星等供应商在推进将逻辑芯片与 DRAM 3D 垂直堆叠的技术，Feynman 可能是首个采用此技术的架构
- **CPO (Co-Packaged Optics)**：光子互联正从交换机级别下沉到 GPU 封装级别

> **影响**：Feynman 仍是蓝图阶段。但它确立了 NVIDIA 的"一年一架构"节奏。从 Feynman 开始，AI 推理性能、延迟和能效将是比训练峰值算力更重要的竞争维度。

---

## 四、关键指标的可视化演进

### 4.1 晶体管数（对数尺度）

```
Fermi(10)    Kepler(12)   Maxwell(14)  Pascal(16)  Volta(17)  Turing(18)
  3B ━━━━━━━ 7.1B ━━━━━━━ 8B ━━━━━━━━ 15.3B ━━━━━ 21.1B ━━━━ 18.6B
    Fermi─────────────────────────────→ Turing: 6.2×（7 年）

Ampere(20)   Ada(22)     Hopper(22)   Blackwell(24)         Rubin(26)
 54.2B ━━━━ 76.3B ━━━━━ 80B ━━━━━━━━━ 208B ━━━━━━━━━━━━━ 336B
    Turing────────────────────────────────→ Rubin: 18×（8 年）

Fermi(10) ←──────────────────────────────────────→ Rubin(26)
   3B                                                336B
                   112× 晶体管增长！
```

### 4.2 显存带宽

```
GDDR5 时代：          HBM 时代：               HBM4 时代：
Fermi:  150 GB/s      P100:   732 GB/s         B200:   8 TB/s
Kepler: 288 GB/s      V100:   900 GB/s         Rubin: 22 TB/s ← 2 万 GB/s!
Maxwell:336 GB/s      A100:  2.0 TB/s
Pascal: 484 GB/s      H100:  3.35 TB/s
(GDDR5X)

Fermi → Rubin：~147× 带宽增长
```

### 4.3 每瓦性能

| 架构 | FP32 TFLOPS | TDP | GF/W |
|------|------------|-----|------|
| Fermi (GTX 580) | 1.58 | 244 W | 6.5 |
| Kepler (K40) | 4.29 | 235 W | 18.3 |
| Maxwell (GTX 980) | 4.61 | 165 W | 27.9 |
| Pascal (P100) | 9.3 | 250 W | 37.2 |
| Volta (V100) | 15.7 | 300 W | 52.3 |
| Turing (2080Ti) | 13.4 | 260 W | 51.5 |
| A100 | 19.5 | 400 W | 48.8 |
| H100 | 67.0 | 700 W | 95.7 |

> **注意**：FP32 TFLOPS/Watt 只反映传统计算效率。加入 Tensor Core 和更低精度后，实际 AI 负载的每瓦性能提升远超此表。

---

## 五、关键技术创新的"首次出现"时间线

```
2010 ─ Fermi:      ECC、C++ GPU、完整 FP64、Unified Addressing
2012 ─ Kepler:     SMX、Hyper-Q、Dynamic Parallelism、GPU Boost
2014 ─ Maxwell:    SMM 四分区、Dedicated SMem、Tiled Caching
2016 ─ Pascal:     NVLink 1.0、HBM2、16nm FinFET、Compute Preemption
2017 ─ Volta:      ██ Tensor Core V1（矩阵乘法硬件）██
2018 ─ Turing:     ██ RT Core（实时光追硬件）██、Mesh Shader、VRS
2020 ─ Ampere:     TF32、MIG、结构化稀疏、Async Copy
2022 ─ Ada:        SER、DLSS 3、第4代 Tensor Core (FP8)、Displaced Micro-Mesh
2022 ─ Hopper:     Transformer Engine、TMA、HBM3、DPX 指令
2024 ─ Blackwell:  ██ Dual-Die 封装、FP4 Tensor Core ██、RAS Engine
2026 ─ Rubin:      ██ HBM4、3nm、100%液冷、NVLink 6 ██
2028 ─ Feynman:    █ 下一代 HBM、推测 3D 堆叠/光子互联 █
```

---

## 六、从演进中看到的趋势

### 6.1 SM 微架构：从"大一统"到"模块化"

```
Fermi:  1 个 SM = 32 CUDA Cores，统一调度
Kepler: SMX = 192 CUDA Cores，4 Warp Scheduler 共享
Maxwell: SMM = 128 CUDA Cores，分 4 个独立子分区 ← 模块化的关键一步
Pascal:  继承 Maxwell，分 GP100(64/SM) 和 GP102(128/SM) 两条线
Volta:   64 CUDA + 8 Tensor Core/SM，Tensor Core 属于每个子分区
Turing:  + RT Core/SM + 并发 INT32/FP32
Ampere:  128 CUDA + 4 Tensor Core + 1 RT Core/SM（消费级）
```

**趋势**：SM 不再是单一的计算单元，而是多个异构加速器（CUDA Core + Tensor Core + RT Core）的**组合体**。每个 SM 的内部设计越来越像一个小型 SoC。

### 6.2 精度演进：从 FP64 到 FP4

```
Fermi:  FP64（科学计算立身之本）
Volta:  FP16 Tensor Core（AI 训练标准）
Ampere: TF32、BF16（训练精度分化）
Ada/Hopper: FP8（训练+推理）
Blackwell: FP4（推理极限压缩）
Rubin:  NVFP4（自适应精度的 FP4）

目标函数已经从"最高精度"变为"给定精度下最大化吞吐"。
```

### 6.3 内存革命：GDDR → HBM → HBM4

```
2010-2014: GDDR5（256-bit bus，< 300 GB/s）
2016:      HBM2（1024-bit per stack，>700 GB/s）—— 3D 堆叠首次出现
2022:      HBM3（H100，3.35 TB/s）
2024:      HBM3e（B200，8 TB/s）
2026:      HBM4（Rubin，22 TB/s）—— 2048-bit per stack，带宽再次翻倍
2028:      HBM4e/HBM5（Feynman）—— 下一跳
```

### 6.4 系统形态：从单卡到整机架

```
2017 (V100):  1卡独立运行，或通过 NVLink Bridge 2/4/8卡
2020 (A100):  DGX A100，8卡一体机，NVSwitch 全互联
2022 (H100):  DGX H100 SuperPOD，256 GPU
2024 (B200):  GB200 NVL72，72 GPU 封装为统一内存域
2026 (Rubin): NVL144 CPX，144 GPU；NVL576，576 GPU
2030+ (?):    整数据中心即一台计算机
```

---

## 七、小结

NVIDIA GPU 架构的十五年演进，可以提炼为以下逻辑：

1. **Fermi 证明 GPU 能算（ECC、FP64）** — 科学计算入场券
2. **Kepler-Maxwell-Pascal 让 GPU 算得更快更好（SM 演进 + NVLink + HBM）** — 为 AI 准备的基础设施
3. **Volta 定义了 AI 计算的范式（Tensor Core）** — "GPU 不只是向量机，是矩阵机"
4. **Turing 将专用加速带入消费级（RT Core + DLSS）** — 图形 + AI 双引擎
5. **Ampere-Ada-Hopper 是 Volta→Blackwell 的迭代** — 精度下降（TF32→FP8），带宽上升
6. **Blackwell-Rubin-Feynman 标志着"后单芯片时代"** — Dual-Die → 3nm → 下一代 HBM → 光子互联

> **本文是该系列关于 GPU 架构代际演进的完整参考**。关于每代架构的 SM 微架构、内存系统和编程模型的更多细节，请参阅系列中对应的分篇笔记。
