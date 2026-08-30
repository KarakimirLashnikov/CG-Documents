# 实时渲染系统 — RDG 与 FrameGraph 概念辨析

> 本文档是「实时渲染系统」系列文档的第 **7** 篇，澄清 RDG 与 FrameGraph 的概念关系，辨析行业术语与引擎专属命名的区别。

---

## 1. 用户核心疑问背景

在学习渲染引擎架构时，常遇到以下困惑：

1. **RDG 和 FrameGraph 是同一个东西吗？**
2. **为什么 UE4/5 称之为 RDG 而其他引擎称之为 FrameGraph？**
3. **帧图是静态的还是动态的？是否在引擎初始化时就固定了？**
4. **自研引擎应该用哪个术语？**

本篇文档将逐一澄清这些问题。

---

## 2. 通用定义

### 2.1 FrameGraph（FG）

**FrameGraph** 是**行业通用术语**，指代以下概念：

> 每帧动态构建的渲染依赖有向无环图（DAG），自动管理资源生命周期、内存屏障、布局转换与执行顺序。

**核心特征**：

| 特征 | 说明 |
|------|------|
| **动态构建** | 每帧根据场景状态重新构建，不是引擎初始化时固定的静态图 |
| **声明式** | Pass 声明依赖与资源需求，系统自动推导执行计划 |
| **自动优化** | 无效 Pass 剔除、资源别名复用、屏障自动推导 |
| **DAG 结构** | 有向无环图保证执行顺序无环路 |

**行业使用案例**：

| 引擎/项目 | 术语名称 | 实现特色 |
|---------|---------|---------|
| Frostbite (EA) | **FrameGraph** | 声明式 Pass + 资源别名复用 |
| Filament (Google) | **FrameGraph** | 轻量实现，移动端优化 |
| Unity HDRP | **Render Graph** | 与 Scriptable Render Pipeline 集成 |
| The Forge | **Frame Graph** | 跨平台实现 |
| 学术论文 | **Render Graph / Frame Graph** | 通用术语 |

### 2.2 RDG（Render Dependency Graph）

**RDG** 是 **UE4/5 的专属命名**：

> Unreal Engine 4/5 中对渲染依赖图的内部命名，本质实现与 FrameGraph 完全一致。

**RDG 在 UE 中的设计**：

```cpp
// UE5 RDG Pass 声明示例 (简化)
FRDGPass* AddPass(
    FRDGBuilder& GraphBuilder,
    const TCHAR* PassName,
    FRDGPassSetupFunc SetupFunc,
    FRDGPassExecuteFunc ExecuteFunc)
{
    // SetupFunc: 声明资源读写依赖
    // ExecuteFunc: 实际GPU命令录制
    // GraphBuilder: DAG构建器，自动编排
}
```

**RDG 核心特征**（与 FrameGraph 一致）：

| 特征 | RDG 实现 | FrameGraph 实现 |
|------|---------|----------------|
| 动态构建 | 每帧根据场景重建 | 每帧根据场景重建 |
| 声明式Pass | SetupFunc声明依赖 | DeclareDependencies声明 |
| 自动屏障 | RDG自动推导屏障 | FrameGraph自动推导 |
| 资源别名 | RDG Texture/Buffer Pool复用 | FrameGraph Alias复用 |
| DAG结构 | FRDGBuilder构建DAG | FrameGraph::BuildDAG |
| 剔除优化 | RDG剔除无效Pass | FrameGraph剔除无效Pass |

---

## 3. 静态模板 vs 动态实例

### 3.1 静态模板（Pass 注册库）

引擎初始化时注册所有**标准 Pass 配置模板**，这些模板定义了 Pass 的基本信息：

```cpp
// 静态模板: 引擎初始化时注册
// 不是"固定帧图"，而是"Pass候选池"

PassTemplate gbufferTemplate = {
    name: "GBuffer",
    pipelineType: GRAPHICS_MESH,
    shaderPath: "gbuffer_mesh",
    requiredResources: [DepthBuffer, SceneMeshes, MaterialTextures],
    outputResources: [GBufferAlbedo, GBufferNormal, GBufferORM, GBufferMotion],
    conditional: false  // 基础Pass始终可能执行
};

PassTemplate ssrTemplate = {
    name: "SSR",
    pipelineType: COMPUTE,
    shaderPath: "ssr_compute",
    requiredResources: [GBufferDepth, GBufferNormal, LightingResult],
    outputResources: [SSRReflection],
    conditional: true   // 条件Pass, 仅在启用SSR时执行
};

PassTemplate rtReflectionTemplate = {
    name: "RTReflection",
    pipelineType: RAY_TRACING,
    shaderPath: "rt_reflection",
    requiredResources: [GBuffer, TLAS, BLAS],
    outputResources: [RTReflectionResult],
    conditional: true   // 仅在启用光追反射时执行
};
```

**关键理解**：静态模板是**候选池**，不是执行计划。模板注册后，引擎知道有哪些 Pass 可用，但**并不决定每帧执行哪些**。

### 3.2 动态实例（每帧帧图）

每帧根据**画面设置、场景开关**，从候选池中选择活跃 Pass，动态构建专属执行图：

```
帧 1: 低画质, 无光追
  活跃Pass: GBuffer + ShadowMap + Lighting + TAA + ToneMapping
  帧图: 5个节点的DAG

帧 2: 高画质, 有光追反射
  活跃Pass: GBuffer + ShadowMap + Lighting + RT-Reflection + TAA + Bloom + ToneMapping
  帧图: 7个节点的DAG (新增RT-Reflection和Bloom)

帧 3: 室内场景, 大量点光源
  活跃Pass: GBuffer + ShadowMap×6 + Lighting + SSR + TAA + Bloom + ToneMapping
  帧图: 8个节点 (ShadowMap实例化为6个光源变体)
```

**每帧帧图构建流程**：

```
场景配置 ──▶ Pass条件判定 ──▶ 从模板池选择活跃模板 ──▶ 声明依赖 ──▶ 构建DAG ──▶ 编译优化 ──▶ 执行
```

### 3.3 对比总结

| 维度 | 静态模板 | 动态实例 |
|------|---------|---------|
| **时机** | 引擎初始化 | 每帧动态 |
| **内容** | Pass配置信息（Shader/管线/资源类型） | 实际执行的DAG（根据场景配置筛选） |
| **是否固定** | 预定义，不随帧变化 | 随帧动态变化 |
| **作用** | 提供Pass候选池 | 生成实际执行计划 |
| **类似概念** | 函数定义 | 函数调用（不同参数不同行为） |

---

## 4. RDG 并非"引擎内置固定图"

### 4.1 常见误区

> ❌ "RDG 是 UE 引擎内部固定的渲染管线图，初始化时就确定了"

这是错误的。RDG（以及所有 FrameGraph 实现）都是**每帧动态构建**的：

```cpp
// UE5 每帧构建RDG的流程 (简化)
void FDeferredShadingRenderer::Render(FRDGBuilder& GraphBuilder) {
    // 每帧重新构建RDG
    // 根据视图配置动态添加Pass

    if (ViewFamily.EngineShowFlags.Shadow) {
        AddShadowPasses(GraphBuilder);  // 动态添加阴影Pass
    }

    if (ViewFamily.EngineShowFlags.SSR) {
        AddSSRPass(GraphBuilder);       // 动态添加SSR Pass
    }

    if (ViewFamily.EngineShowFlags.RayTracing) {
        AddRayTracingPasses(GraphBuilder);  // 动态添加光追Pass
    }

    // 编译+执行RDG (动态构建的帧图)
    GraphBuilder.Compile();
    GraphBuilder.Execute();
}
```

### 4.2 UE RDG 的动态性证据

- **ViewFamily.EngineShowFlags**：每帧可切换渲染特性，RDG 响应式重建
- **画质档位切换**：从低画质切换到高画质时，RDG 自动增删 Pass
- **场景内容变化**：进入室内场景 → 增加局部光源 Pass；进入室外 → 增加大气 Pass
- **硬件能力适配**：检测 GPU 是否支持光追 → 动态决定是否添加光追 Pass

---

## 5. 自研引擎命名建议

### 5.1 结论

> **自研系统统一使用 FrameGraph 命名更通用**。

**理由**：

1. **行业通用性**：FrameGraph 是学术界和行业广泛使用的标准术语，RDG 仅是 UE 内部命名
2. **概念清晰**：FrameGraph 直接表达"帧级图"的含义，RDG 的"Dependency Graph"概念更泛化
3. **避免品牌绑定**：使用 RDG 容易让读者误以为采用了 UE 的特定实现
4. **国际化友好**：FrameGraph 是英文通用术语，便于跨团队交流

### 5.2 命名对照表

| 自研术语 | 建议命名 | UE对应 | 说明 |
|---------|---------|--------|------|
| 帧图系统 | **FrameGraph** | RDG (Render Dependency Graph) | 统一使用FrameGraph |
| 帧 | **Pass** | FRDGPass | 保持Pass命名 |
| 虚拟资源句柄 | **VirtualResource / ResourceHandle** | FRDGTexture / FRDGBuffer | 资源抽象统一 |
| 帧图构建器 | **FrameGraphBuilder** | FRDGBuilder | 构建器命名 |
| 帧图编译 | **Compile** | Compile | 保持一致 |
| 别名复用 | **Alias / MemoryAlias** | RDG Pool Aliasing | 别名复用命名 |
| 屏障推导 | **BarrierDerivation** | RDG Automatic Barriers | 屏障自动推导 |

### 5.3 不建议使用的命名

| 不建议命名 | 原因 |
|-----------|------|
| RDG | UE专属命名，概念泛化，容易与UE绑定混淆 |
| RenderPipeline | 太泛化，与Vulkan Pipeline概念冲突 |
| RenderingGraph | 不够精简，FrameGraph更简洁 |
| DAG | 太底层，仅描述数据结构而非渲染概念 |

---

## 6. 底层设计逻辑一致性

无论命名为 FrameGraph 还是 RDG，底层设计逻辑**完全一致**：

```
共同的设计逻辑 (FrameGraph = RDG = Render Graph):

1. 声明式Pass注册
   ├─ 每个Pass声明: 管线类型 + Shader + 资源读写 + 附件
   └─ 不硬编码物理资源(VkImage/VkBuffer)

2. DAG动态构建
   ├─ 根据依赖关系构建有向无环图
   └─ 支持条件Pass动态增删

3. 编译优化
   ├─ 无效Pass剔除
   ├─ 资源生命周期计算 + 别名复用
   ├─ 自动屏障/布局推导
   └─ Subpass合并(移动端)

4. 执行编排
   ├─ 多线程CommandBuffer录制
   ├─ 多队列并行提交
   └─ 帧延迟资源回收

5. 收尾管理
   ├─ 描述符池重置
   ├─ 延迟释放队列处理
   └─ 环形缓冲回收
```

---

## 7. 概念辨析总结

| 问题 | 答案 |
|------|------|
| RDG和FrameGraph是同一个东西吗？ | **是**。RDG是UE对FrameGraph的专属命名，本质完全一致 |
| 为什么UE称之为RDG？ | UE内部命名偏好，"Dependency Graph"强调依赖关系视角 |
| 帧图是静态的吗？ | **否**。静态模板仅注册Pass候选池，每帧动态构建实际帧图 |
| 自研引擎应该用哪个术语？ | **FrameGraph**，更通用、更清晰、避免UE绑定 |
| RDG是否是UE内置固定图？ | **否**。每帧动态构建，响应场景配置和硬件能力 |

---

*上一篇：[06-FrameGraph帧图系统](06-FrameGraph帧图系统.md)*
*下一篇：[08-进阶技术拓展](08-进阶技术拓展.md)*
