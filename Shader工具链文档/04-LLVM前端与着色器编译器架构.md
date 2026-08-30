# LLVM 前端与着色器编译器架构

> 系列第 **4** 篇。很多人以为"LLVM 跟 shader 没关系"——恰恰相反：DXC 是 LLVM/Clang 的分支，LLVM 主线早已内置 SPIR-V 后端，Mesa 的软件光栅化器靠 LLVM JIT。本文讲清 LLVM 在着色器工具链中的位置、三条到 SPIR-V 的路径，并动手写 Pass 与前端工具。

---

## 1. 为什么 shader 工具链要谈 LLVM

### 1.1 三个层次的关系

```
┌──────────────────────────────────────────────────────────┐
│  误区：LLVM 是 CPU 编译器，跟 shader 无关                   │
├──────────────────────────────────────────────────────────┤
│  事实 1：DXC（DirectX Shader Compiler）本身就是             │
│          LLVM 3.7 + Clang 3.7 的一个长期分支                │
│  事实 2：LLVM 主线自 2021 年起内置 SPIR-V 后端               │
│          （experimental，2025 年有转正 RFC）                 │
│  事实 3：Mesa 的 llvmpipe / lavapipe、SwiftShader 都靠       │
│          LLVM JIT 生成软件光栅化代码                         │
│  事实 4：DXIL 就是"LLVM 3.7 bitcode + D3D12 规则约束"        │
└──────────────────────────────────────────────────────────┘
```

### 1.2 LLVM 在工具链中的三种角色

| 角色 | 说明 | 代表 |
|------|------|------|
| **① 作为编译器底座** | 复用 LLVM 的优化器与代码生成器 | DXC（DXIL 路径）、LLVM SPIR-V 后端 |
| **② 作为前端工具库** | 复用 Clang 的词法/语法/语义分析做 shader 静态分析、重构、代码生成 | Clang LibTooling 自研 shader 工具 |
| **③ 作为运行时 JIT** | 把 IR 即时编译成本机代码 | llvmpipe、lavapipe、SwiftShader、Embree |

### 1.3 什么时候你真的需要碰 LLVM

| 需求 | 是否要 LLVM |
|------|------------|
| 用反射生成描述符布局 | ❌ 完全不需要（SPIRV-Reflect 足够） |
| 写一门自己的着色器 DSL | ✅ 需要（复用 Clang 或自己造前端） |
| 对 shader 做自定义优化/插桩 | ⚠️ 可以做在 LLVM IR 层（若走 LLVM 路径），也可以做在 SPIR-V 层（SPIRV-Tools，更通用） |
| 编译 OpenCL / SYCL 到 SPIR-V | ✅ 直接用 LLVM SPIR-V 后端 |
| 让 C++ 跑在 GPU 上（CUDA 风格 kernel） | ✅ LLVM SPIR-V 后端 / DPC++ |
| 跨平台转译 | ❌ 用 SPIRV-Cross |

---

## 2. LLVM 编译管线全景

```
源码
 │
 ▼
┌──────────────┐   词法单元流
│ 1. Lexer     │ ─────────────▶ Token Stream
└──────────────┘
┌──────────────┐   语法树（含错误恢复）
│ 2. Parser    │ ─────────────▶ AST
└──────────────┘
┌──────────────┐   类型检查、名字解析、隐式转换
│ 3. Sema      │ ─────────────▶ 带类型的 AST
└──────────────┘
┌──────────────┐
│ 4. CodeGen   │ ─────────────▶ LLVM IR（SSA + 显式 CFG）
└──────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────┐
│ 5. Optimization Pipeline（-O0 ~ -O3, -Os, -Oz）        │
│    InstCombine / GVN / LICM / SLP Vectorizer / Inline │
│    ★ 目标无关优化（Target-independent）                │
└──────────────────────────┬───────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────┐
│ 6. Target CodeGen（SelectionDAG 或 GlobalISel）        │
│    IR → SelectionDAG/Generic MIR → 合法化 → 寄存器分配 │
│    → 指令调度 → MC / 目标文件                          │
│    ★ 目标相关（SPIR-V 后端就插在这里）                  │
└──────────────────────────────────────────────────────┘
```

**关键点**：SPIR-V 后端位于第 6 步（CodeGen），因此**前面所有目标无关优化都免费获得**——这是 LLVM 路径最大的吸引力。

> ⚠️ 但要注意：SPIR-V 是"逻辑 IR"而非"机器码"，它没有寄存器、没有指令编码。所以 SPIR-V 后端实际做的是 **IR → SPIR-V 指令的选择与发射**，而不是传统的寄存器分配 + 汇编。它基于 **GlobalISel** 流程。

---

## 3. Clang 前端架构拆解

Clang 是 LLVM 生态的 C/C++/Objective-C 前端，也是**自研 shader 前端最值得抄的对象**。

### 3.1 分层

```
┌─────────────────────────────────────────────────────────┐
│ Driver（clang 命令行）                                     │
│   解析参数、决定 -cc1 调用、管理多阶段编译                   │
├─────────────────────────────────────────────────────────┤
│ Frontend（CompilerInstance / FrontendAction）              │
│   管理 SourceManager、FileManager、DiagnosticsEngine       │
├─────────────────────────────────────────────────────────┤
│ Lexer / Preprocessor                                      │
│   字符流 → Token；宏展开、#include 处理                     │
├─────────────────────────────────────────────────────────┤
│ Parser                                                    │
│   Token → AST；递归下降 + 错误恢复                          │
├─────────────────────────────────────────────────────────┤
│ Sema（语义分析）                                            │
│   ★ 由 Parser 回调驱动：类型检查、重载决议、名字查找          │
│   ★ AST 节点在这里被"补全/润色"                            │
├─────────────────────────────────────────────────────────┤
│ AST                                                        │
│   Decl / Stmt / Expr / Type 四大类，可序列化、可遍历          │
├─────────────────────────────────────────────────────────┤
│ CodeGen（clang/lib/CodeGen）                               │
│   AST → LLVM IR                                            │
└─────────────────────────────────────────────────────────┘
```

### 3.2 AST 的四大节点族

| 族 | 代表类 | 说明 |
|----|--------|------|
| `Decl` | `FunctionDecl`、`VarDecl`、`RecordDecl`、`ParmVarDecl` | 声明 |
| `Stmt` | `IfStmt`、`ForStmt`、`ReturnStmt`、`CompoundStmt` | 语句 |
| `Expr` | `BinaryOperator`、`CallExpr`、`MemberExpr`、`CastExpr` | 表达式 |
| `Type` | `BuiltinType`、`RecordType`、`PointerType`、`ConstantArrayType` | 类型（**有类型系统保证，非字符串**） |

> **与 SPIR-V 反射的对比**：AST 层能拿到"源码级"信息（函数名、参数名、注释、模板实参），SPIR-V 反射拿不到。但 AST 是编译期才存在的，运行时不可用。二者**互补**：AST 工具负责"生成/检查"，反射负责"运行时消费"。

---

## 4. 三条到 SPIR-V 的路径

```
┌────────────────────────────────────────────────────────────────┐
│ 路径 ①：glslang —— 无 LLVM                                       │
│                                                                  │
│   GLSL / HLSL → 词法语法分析 → glslang AST → 直接发射 SPIR-V       │
│                                                                  │
│   ✓ 轻量、快、官方参考实现                                         │
│   ✗ 拿不到 LLVM 的优化器                                           │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ 路径 ②：DXC —— LLVM 血统，但 SPIR-V 不走 LLVM IR                  │
│                                                                  │
│   HLSL → Clang 前端 → HLSL AST (HLModule)                        │
│             │                                                    │
│             ├──[DXIL 路径]──▶ Clang CodeGen → LLVM 3.7 IR        │
│             │                   → DxilGeneration Passes → DXIL    │
│             │                                                    │
│             └──[SPIR-V 路径]─▶ SpirvEmitter 直接遍历 AST          │
│                                 → SPIR-V                          │
│                                 ★ 不经过 LLVM IR！                │
│                                                                  │
│   ✓ HLSL 一等支持；-fspv-* 选项丰富                                │
│   ✗ SPIR-V 路径无法复用 LLVM 优化器（优化靠 spirv-opt 补）          │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ 路径 ③：LLVM SPIR-V 后端（主线）+ SPIRV-LLVM-Translator           │
│                                                                  │
│   C/C++/OpenCL/SYCL → Clang → LLVM IR                            │
│       → [LLVM SPIR-V 后端: GlobalISel] → SPIR-V                  │
│   或 → [llvm-spirv 转换器]            → SPIR-V                   │
│                                                                  │
│   ✓ 复用 LLVM 全部优化；C++ 能直接跑 GPU                           │
│   ✗ 图形着色器（VS/PS/GS 等）支持仍在推进中                        │
└────────────────────────────────────────────────────────────────┘
```

### 4.1 路径对比

| 维度 | glslang | DXC | LLVM SPIR-V 后端 |
|------|---------|-----|-----------------|
| 输入语言 | GLSL、HLSL（部分） | HLSL | C/C++、OpenCL、SYCL |
| 是否经 LLVM IR | ❌ | ❌（SPIR-V 路径） | ✅ |
| 优化来源 | `spirv-opt`（外挂） | `spirv-opt`（外挂） | LLVM 自身 + `spirv-opt` |
| 图形着色器支持 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐（推进中） |
| 计算内核支持 | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐（OpenCL 3.0 一致性认证） |
| Vulkan 目标适配 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐（`-fspv-target-env`） | ⭐⭐⭐ |
| 典型使用 | `--target-env vulkan1.3` | `-spirv -fspv-target-env=vulkan1.3` | `-target spirv64` |

### 4.2 DXC 常用选项（实战）

```bash
dxc -T ps_6_6 -E PSMain \
    -spirv \
    -fspv-target-env=vulkan1.3 \
    -fvk-b-shift 20 all \          # 所有 CBV 的 binding 左移 20 位
    -fvk-s-shift 10 all \          # 采样器
    -fvk-t-shift 0  all \          # SRV
    -fvk-u-shift 30 all \          # UAV
    -fvk-use-dx-layout \           # 按 HLSL 打包规则（与 D3D12 一致）
    -enable-16bit-types \
    -fspv-reflect \                # 生成反射辅助信息
    -fspv-debug=vulkan-with-source \
    -H \                           # 人类可读
    -Fo pbr.frag.spv pbr.hlsl
```

**HLSL 中的 Vulkan 专属属性**（DXC 支持）：

```hlsl
[[vk::binding(3, 1)]]        Texture2D    g_Albedo;      // set=1, binding=3
[[vk::push_constant]]        cbuffer PerDraw { float4x4 model; };
[[vk::constant_id(0)]]       const bool  ENABLE_TAA = true;
[[vk::location(0)]]          float4      PSMain([[vk::location(0)]] float2 uv) : SV_Target;
[[vk::combinedImageSampler]] Texture2D    g_Combined;     // 强制合并 image+sampler
```

> **重要**：有了 `[[vk::binding(set, binding)]]`，你就能在**源码里**直接声明 set/binding，SPIRV-Reflect 读到的就是最终值，不需要运行时 `Change*` 重映射。这是最省心的方案。

### 4.3 LLVM SPIR-V 后端现状（截至 2025–2026）

根据 LLVM 社区 RFC《Promoting SPIR-V to an official target》与官方 `SPIRVUsage` 文档：

| 项目 | 状态 |
|------|------|
| 引入时间 | 2021 年 3 月，作为 **experimental** target |
| 转正进程 | 已有正式 RFC 提案转为 official target |
| 计算路径 | **OpenCL 3.0 一致性认证**；SYCL CTS 通过率 ~99% |
| 已实现扩展 | 31 个 SPIR-V 扩展（持续增长） |
| 图形路径 | "Vulkan and HLSL support" 为 **ongoing work** |
| 维护方 | AMD、Google、Intel、Microsoft 等多组织 |
| 社区 | SPIR-V Backend Working Group 双周会议（公开） |
| Khronos 表态 | 在其 RFP 中表示该后端"预期将完全取代 translator" |
| 下游采用 | Intel 计划在 SYCL/DPC++ 与 XPU Triton 编译流中使用；Microsoft 在 DirectX 12 中采用 SPIR-V |

> **实践含义**：如果你的目标是 **GPU 计算**（OpenCL/SYCL/C++ kernel），LLVM SPIR-V 后端已经可用。如果目标是 **图形着色器**，现阶段仍应用 glslang / DXC。

---

## 5. SPIR-V 在 LLVM IR 中的表示

这是理解 LLVM → SPIR-V 的关键：SPIR-V 的很多概念（存储类、图像类型、装饰）在 LLVM IR 里**没有原生对应**，需要**编码**。官方 `SPIRVUsage` 文档把这些归为几类：

### 5.1 特殊类型：Inline SPIR-V Types

用 LLVM 的 **target extension type** 语法表达 SPIR-V 的不透明类型：

```llvm
; 形态（示意，操作数随类型而异）
%image   = type target("spirv.Image",   <sampled-type>, <dim>, <depth>, <arrayed>, <ms>, <sampled>, <format>)
%sampler = type target("spirv.Sampler")
%event   = type target("spirv.Event")
%as      = type target("spirv.AccelerationStructure")
```

好处：**不污染 LLVM 核心类型系统**，且后端能原样还原成 `OpTypeImage` 等指令。

### 5.2 目标内联函数：`llvm.spv.*`

无法用标准 IR 表达的 SPIR-V 操作，用目标内联函数：

```llvm
; Clang 层暴露为 __spirv_* 内建
; IR 层落到目标内联函数
declare i32 @llvm.spv.load(...)
declare void @llvm.spv.store(...)
declare ...  @llvm.spv.image.sample(...)
```

### 5.3 元数据：Decoration 与执行模式

Decoration、BuiltIn、ExecutionMode 等"挂在 ID 上的属性"，在 LLVM IR 里用**命名元数据（named metadata）**表达，形如 `!spirv.*`：

```llvm
!spirv.Source     = !{!0}
!spirv.Decoration = !{!1, !2}
!1 = !{ptr @g_Albedo, i32 33, i32 1}     ; DecorationBinding = 33, value = 1
```

后端发射时会把这些元数据还原成 `OpDecorate` / `OpMemberDecorate` / `OpExecutionMode`。

### 5.4 地址空间 = 存储类

| LLVM addrspace（OpenCL/SPIR 约定） | 语义 |
|---|---|
| 0 | Private（默认） |
| 1 | Global |
| 2 | Constant |
| 3 | Local（工作组共享） |
| 4 | Generic |

```llvm
; OpenCL: __global float* p
%p = alloca ptr addrspace(1)
```

Vulkan 目标下（PushConstant / Uniform / StorageBuffer 等）由后端做 SPIR-V storage class ↔ address space 的映射，具体编号规则见官方 `SPIRVUsage` 文档。

### 5.5 扩展指令集：函数调用

GLSL.std.450 这类扩展指令（如 `fma`、`mix`、`reflect`）在 IR 中表达为**函数调用**：

```llvm
%r = call float @_Z3fmifff(float %a, float %b, float %c)   ; 或 mangled 名
; 后端还原成 OpExtInst %glsl450 %fma
```

### 5.6 完整映射表

| SPIR-V 概念 | LLVM IR 表示 |
|-------------|-------------|
| `OpTypeImage/Sampler/Event/...` | target extension type `target("spirv.X", ...)` |
| `OpDecorate` / `OpMemberDecorate` | 命名元数据 `!spirv.*` |
| `OpExecutionMode` | 命名元数据 + 函数属性 |
| Storage Class | address space + 元数据 |
| `OpExtInst`（GLSL.std.450） | 函数调用 |
| BuiltIn 变量 | 元数据标注 + 特殊 intrinsic |
| 原子操作 | `llvm.spv.atomic.*` / 原子 IR + 元数据 |
| 图像操作 | `llvm.spv.image.*` / 带 target type 的调用 |
| 工作组/子组操作 | `llvm.spv.*` 目标内联函数 |

---

## 6. 动手：用 Clang/llc 编译 SPIR-V

### 6.1 从 OpenCL kernel 出发

```c
// kernel.cl
__kernel void add(__global const float* a,
                  __global const float* b,
                  __global float* c) {
    int gid = get_global_id(0);
    c[gid] = a[gid] + b[gid];
}
```

```bash
# 一条命令出 SPIR-V
clang --target=spirv64 -O1 -c kernel.cl -o kernel.spv

# 或分两步（便于插入自定义 Pass）
clang --target=spirv64 -O1 -S -emit-llvm kernel.cl -o kernel.ll
llc   -O1 -mtriple=spirv64-unknown-unknown kernel.ll -o kernel.spv -filetype=obj

# 校验 + 反汇编
spirv-val --target-env vulkan1.3 kernel.spv
spirv-dis kernel.spv | head -40
```

### 6.2 Triple 的四个部分

```
        spirv64v1.6 - amd - vulkan1.3
        │   │  │      │     │
        │   │  │      │     └─ OS：unknown / vulkan / vulkan1.2 / vulkan1.3 / amdhsa
        │   │  │      └─────── Vendor：unknown / amd
        │   │  └────────────── Subarch：v1.0 ~ v1.6（省略则由后端推断）
        │   └────────────────── Arch：spirv32 / spirv64 / spirv（逻辑布局）
        └────────────────────── 架构
```

```bash
llc -mtriple=spirv64v1.6-amd-amdhsa in.ll -o out.spvt
llc -mtriple=spirv32-unknown-unknown  in.ll -o out.spvt
```

### 6.3 扩展与调试信息

```bash
# 启用扩展
llc -O1 -mtriple=spirv64-unknown-unknown \
    --spirv-ext=+SPV_KHR_non_semantic_info \
    in.ll -o out.spvt

# 发射 NonSemantic.Shader.DebugInfo.100（新版本用 -g）
llc -g --spirv-ext=+SPV_KHR_non_semantic_info in.ll -o out.spvt
# （旧标志 --spv-emit-nonsemantic-debug-info 已废弃，将被移除）
```

> ⚠️ **与反射的关系**：`SPV_KHR_non_semantic_info` 注入的 NonSemantic 调试信息**不在** `OpName` 里，SPIRV-Reflect 不会把它解析成 `name`。若依赖名字做反射，仍需保留 `OpName`（即**不要**完整 strip debug）。

### 6.4 SPIRV-LLVM-Translator（Khronos）

主线后端之外的另一条路：**双向**转换器。

```bash
llvm-spirv      kernel.bc  -o kernel.spv    # LLVM IR → SPIR-V
llvm-spirv -r   kernel.spv -o kernel.bc     # SPIR-V  → LLVM IR（反向）
```

| 对比 | LLVM 主线后端 | SPIRV-LLVM-Translator |
|------|--------------|----------------------|
| 集成方式 | `llc -mtriple=spirv64` | 独立工具 / 库 |
| 方向 | 单向（IR → SPIR-V） | **双向** |
| 依赖 LLVM 版本 | 跟随主线 | 有版本对应表（升级成本高） |
| 下游 | 新兴采用（DPC++ 计划迁移） | 成熟（DPC++/oneAPI 现行方案） |
| 趋势 | Khronos 认为将取代 translator | 逐渐过渡 |

---

## 7. 动手：写一个 LLVM Pass 插件

场景：给所有 SPIR-V kernel 自动插桩（统计执行次数 / 注入调试钩子）。这正是"IR 级元编程"的一种。

### 7.1 代码

```cpp
// ShaderInstrument.cpp
#include "llvm/IR/PassManager.h"
#include "llvm/IR/Module.h"
#include "llvm/IR/Function.h"
#include "llvm/IR/IRBuilder.h"
#include "llvm/Passes/PassBuilder.h"
#include "llvm/Passes/PassPlugin.h"
#include "llvm/Support/raw_ostream.h"

using namespace llvm;

namespace {

struct ShaderInstrumentPass : public PassInfoMixin<ShaderInstrumentPass> {
    PreservedAnalyses run(Module& M, ModuleAnalysisManager&) {
        bool changed = false;

        for (Function& F : M) {
            // 只处理 kernel（OpenCL kernel 的调用约定是 SPIR_KERNEL）
            if (F.getCallingConv() != CallingConv::SPIR_KERNEL) continue;
            if (F.isDeclaration()) continue;

            LLVMContext& Ctx = M.getContext();
            IRBuilder<> B(&*F.getEntryBlock().getFirstInsertionPt());

            // 声明一个外部计数器累加函数
            FunctionCallee Counter = M.getOrInsertFunction(
                "__shader_invocation_count",
                Type::getVoidTy(Ctx),
                PointerType::getUnqual(Ctx));

            // 以函数名作为常量字符串传入
            Value* Name = B.CreateGlobalStringPtr(F.getName());
            B.CreateCall(Counter, { Name });

            // 可选：给函数加一个属性，便于后端识别
            F.addFnAttr("shader.instrumented", "true");
            changed = true;
        }
        return changed ? PreservedAnalyses::none() : PreservedAnalyses::all();
    }

    // 让 -O0 也能运行
    static bool isRequired() { return true; }
};

} // namespace

extern "C" LLVM_ATTRIBUTE_WEAK ::llvm::PassPluginLibraryInfo
llvmGetPassPluginInfo() {
    return {
        LLVM_PLUGIN_API_VERSION,
        "ShaderInstrument",
        LLVM_VERSION_STRING,
        [](PassBuilder& PB) {
            // 在优化管线最早期插入（此时 IR 最接近源码）
            PB.registerPipelineStartEPCallback(
                [](ModulePassManager& MPM, OptimizationLevel) {
                    MPM.addPass(ShaderInstrumentPass());
                });
            // 也可以注册到 clang 的命令行：
            PB.registerPipelineParsingCallback(
                [](StringRef Name, ModulePassManager& MPM,
                   ArrayRef<PassBuilder::PipelineElement>) {
                    if (Name == "shader-instrument") {
                        MPM.addPass(ShaderInstrumentPass());
                        return true;
                    }
                    return false;
                });
        },
    };
}
```

### 7.2 构建与使用

```bash
# 编译成插件
clang++ -shared -fPIC ShaderInstrument.cpp \
    $(llvm-config --cxxflags) -o libShaderInstrument.so

# 使用（两种方式）
clang --target=spirv64 -O1 -fpass-plugin=./libShaderInstrument.so -c kernel.cl -o kernel.spv
opt -load-pass-plugin=./libShaderInstrument.so -passes=shader-instrument in.ll -o out.ll
```

### 7.3 能做 / 不能做

| ✅ 能做 | ❌ 不能做（在 IR 层） |
|--------|---------------------|
| 插桩、计数、埋点 | 修改 SPIR-V 的 `DescriptorSet/Binding`（此时还没生成） |
| 内联控制、循环变换 | 直接写 `OpDecorate`（要走 translator 的元数据约定） |
| 常量传播、特化 | 依赖 SPIR-V 独有语义的优化 |
| 自定义 ABI 变换 | — |

> **结论**：若你的目标是"改绑定号/location"，**在 SPIR-V 层做更直接**（SPIRV-Reflect 的 `Change*` 或 SPIRV-Tools 的自定义 Pass）。若目标是"通用优化/插桩"，LLVM IR 层更强大。

---

## 8. 动手：用 Clang LibTooling 写 shader 前端工具

场景：扫描所有 shader 源码，提取自定义注解，自动生成**绑定注册表**与 C++ 绑定头。这是"元编程 + 反射"的结合点。

### 8.1 代码骨架

```cpp
// ShaderScanner.cpp
#include "clang/AST/ASTConsumer.h"
#include "clang/AST/RecursiveASTVisitor.h"
#include "clang/Frontend/CompilerInstance.h"
#include "clang/Frontend/FrontendAction.h"
#include "clang/Tooling/CommonOptionsParser.h"
#include "clang/Tooling/Tooling.h"
#include "llvm/Support/CommandLine.h"

using namespace clang;
using namespace clang::tooling;

// ---------- 1. 遍历器 ----------
class ShaderVisitor : public RecursiveASTVisitor<ShaderVisitor> {
public:
    explicit ShaderVisitor(ASTContext& Ctx) : Ctx_(Ctx) {}

    // 捕获全局变量（HLSL 中 Texture2D / cbuffer 等通常是全局声明）
    bool VisitVarDecl(VarDecl* D) {
        if (!D->hasGlobalStorage()) return true;

        // 读取自定义注解属性（假设我们定义了 __attribute__((annotate("set:1")))）
        if (D->hasAttrs()) {
            for (const auto* A : D->attrs()) {
                if (const auto* An = dyn_cast<AnnotateAttr>(A)) {
                    llvm::outs() << "resource: " << D->getNameAsString()
                                 << "  annotate=" << An->getAnnotation() << "\n";
                }
            }
        }

        // 类型信息（比 SPIR-V 反射更丰富）
        QualType T = D->getType();
        llvm::outs() << "  type=" << T.getAsString()
                     << "  size=" << Ctx_.getTypeSizeInChars(T).getQuantity() << "\n";
        return true;
    }

    // 捕获入口点函数
    bool VisitFunctionDecl(FunctionDecl* D) {
        if (!D->isThisDeclarationADefinition()) return true;
        llvm::outs() << "function: " << D->getNameAsString()
                     << " params=" << D->getNumParams() << "\n";
        return true;
    }

private:
    ASTContext& Ctx_;
};

// ---------- 2. Consumer ----------
class ShaderConsumer : public ASTConsumer {
public:
    void HandleTranslationUnit(ASTContext& Ctx) override {
        ShaderVisitor V(Ctx);
        V.TraverseDecl(Ctx.getTranslationUnitDecl());
    }
};

// ---------- 3. FrontendAction ----------
class ShaderFrontendAction : public ASTFrontendAction {
public:
    std::unique_ptr<ASTConsumer> CreateASTConsumer(CompilerInstance&, StringRef) override {
        return std::make_unique<ShaderConsumer>();
    }
};

// ---------- 4. main ----------
static llvm::cl::OptionCategory ToolCategory("shader-scanner");

int main(int argc, const char** argv) {
    auto Expected = CommonOptionsParser::create(argc, argv, ToolCategory);
    if (!Expected) { llvm::errs() << toString(Expected.takeError()); return 1; }

    ClangTool Tool(Expected->getCompilations(), Expected->getSourcePathList());
    // 追加 shader 特有的宏/选项
    Tool.appendArgumentsAdjuster(getInsertArgumentAdjuster(
        {"-x", "hlsl", "-D", "SHADER_SCANNER=1"}, ArgumentInsertPosition::BEGIN));
    return Tool.run(newFrontendActionFactory<ShaderFrontendAction>().get());
}
```

### 8.2 CMake 集成

```cmake
find_package(LLVM REQUIRED CONFIG)
find_package(Clang REQUIRED CONFIG)

add_executable(shader-scanner ShaderScanner.cpp)
target_include_directories(shader-scanner PRIVATE ${LLVM_INCLUDE_DIRS} ${CLANG_INCLUDE_DIRS})
target_compile_definitions(shader-scanner PRIVATE ${LLVM_DEFINITIONS})
target_link_libraries(shader-scanner PRIVATE clangTooling clangFrontend clangAST clangBasic)

# 或者用 Clang 的静态库集合
# clang_target_link_libraries(shader-scanner PRIVATE clangTooling)
```

### 8.3 这类工具能做什么

| 工具 | 作用 |
|------|------|
| **绑定注册表生成器** | 扫描注解 → 生成 `shader_bindings.json` / `.h`，供反射层对照 |
| **语义标注器** | 把 `PerFrame` / `PerView` / `PerMaterial` 语义写进元数据（SPIR-V 里没有的！） |
| **变体清单生成器** | 分析所有 `#ifdef` / 模板实参，产出需要编译的 permutation 列表 |
| **跨阶段校验器** | 检查 VS 输出与 PS 输入的 location/类型是否匹配（在编译前就发现问题） |
| **文档生成器** | 从 AST 生成 shader 接口文档 |

> ⚠️ **现实约束**：Clang 只能解析 C/C++/OpenCL。**HLSL 不是 Clang 原生支持的**。要用 Clang 分析 HLSL，需要 DXC 的 Clang 分支（`dxc` 自带的 `clang` 库）或自己做语言扩展。多数团队退而求其次：**用正则/自定义解析器扫源码**，或者干脆让 **Slang** 干这件事（它有完整的语言服务与反射）。

---

## 9. DXIL vs SPIR-V：两个 LLVM 世界的对照

| 维度 | DXIL | SPIR-V |
|------|------|--------|
| 本质 | LLVM 3.7 bitcode + D3D12 约束 | Khronos 独立二进制 IR |
| 格式 | LLVM bitcode（.ll / .bc 结构） | 自定义字流（见 [01 篇](01-SPIR-V模块结构与反射原理.md)） |
| 校验 | `dxil.dll`（D3D12 运行时签名验证） | `spirv-val` |
| 反射 | `ID3D12ShaderReflection`（读 DXIL 元数据） | SPIRV-Reflect / SPIRV-Cross |
| 优化时机 | 驱动可在运行时再优化 | Vulkan 下通常由 `spirv-opt` 离线优化 |
| 可读性 | `llvm-dis` | `spirv-dis` |
| 生态 | 仅 D3D12 | Vulkan / OpenCL / SYCL |

> **为什么 DXIL 要锁死 LLVM 3.7**：DXIL 是**驱动的输入契约**，若 LLVM 版本升级导致 bitcode 格式变化，所有驱动都要更新。所以 DXIL 冻结在 LLVM 3.7 + 一套自定义元数据约定。这也是"用 LLVM IR 做分发格式"的经典代价。SPIR-V 作为**独立标准**则没有这个包袱。

---

## 10. LLVM 在软件渲染中的角色

| 项目 | LLVM 的角色 |
|------|------------|
| **llvmpipe**（Mesa GL 软件光栅化） | 顶点/片元着色器 → LLVM IR → JIT 成本机代码，多线程分块执行 |
| **lavapipe**（Mesa Vulkan 软实现） | 基于 llvmpipe，实现 Vulkan 1.x 软渲染 |
| **SwiftShader**（Google） | 用 LLVM（Reactor JIT）实现高性能 Vulkan/GL 软渲染 |
| **Mesa 硬件驱动**（历史） | i965 曾用 LLVM 做后端；现代 Intel（iris）/ AMD（ACO）已改用 NIR → 原生 ISA，不再依赖 LLVM |
| **Embree / OptiX 类** | LLVM JIT 生成光线求交内核 |

**实践价值**：在没有 GPU 的 CI 环境里，用 lavapipe + LLVM JIT 跑渲染测试，是**验证 shader 工具链正确性**的常用手段。

---

## 11. 限制与坑

| # | 问题 | 说明 | 对策 |
|---|------|------|------|
| 1 | 图形着色器支持不完整 | LLVM SPIR-V 后端的 Vulkan/HLSL 支持是 ongoing work | 图形路径用 glslang/DXC |
| 2 | HLSL 不是 Clang 原生语言 | 想用 LibTooling 分析 HLSL 需要 DXC 的 Clang 分支 | 改用 Slang，或自研轻量解析器 |
| 3 | LLVM 版本耦合 | Translator 有严格的 LLVM 版本对应表 | 用主线后端，或锁定 LLVM 版本 |
| 4 | 构建体积与耗时 | 完整 LLVM 是 GB 级构建 | 只链接需要的组件（clangTooling / 目标后端） |
| 5 | IR 层拿不到 SPIR-V 装饰 | Decoration 此时还是元数据 | 要改绑定，去 SPIR-V 层做 |
| 6 | 优化可能打乱调试信息 | `-O2` 后行号映射失真 | 调试构建用 `-O0 -g` |
| 7 | NonSemantic 与 OpName 不互通 | 注入 NSDI 不会让 SPIRV-Reflect 的 `name` 出现 | 保留 `OpName` |
| 8 | 插件 ABI 脆弱 | Pass 插件必须与 LLVM 版本、编译选项严格一致 | 用同一套 `llvm-config` 产出 |

---

## 12. 小结

| 主题 | 结论 |
|------|------|
| LLVM 与 shader 的关系 | DXC 是 LLVM 分支；LLVM 主线有 SPIR-V 后端；软渲染靠 LLVM JIT |
| 三条 SPIR-V 路径 | glslang（无 LLVM，图形最强）/ DXC（HLSL 一等，SPIR-V 走 AST）/ LLVM 后端（计算最强，图形推进中） |
| SPIR-V 在 IR 中的表示 | target extension type + `llvm.spv.*` 内联 + `!spirv.*` 元数据 + address space |
| 动手方向 | Pass 插件（`llvmGetPassPluginInfo`）做 IR 级插桩/优化；LibTooling 做前端分析与代码生成 |
| 与反射的分工 | **IR 层负责生成与变换，SPIR-V 层负责事实，反射层负责消费** |
| 选型建议 | 图形 → glslang/DXC；计算 → LLVM SPIR-V 后端；跨平台 → SPIRV-Cross |

---

*上一篇：[03-反射方案对比与选型](03-反射方案对比与选型.md) | 下一篇：[05-可编程着色器元编程](05-可编程着色器元编程.md)*
