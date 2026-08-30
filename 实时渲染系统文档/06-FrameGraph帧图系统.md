# 实时渲染系统 — FrameGraph 帧图系统

> 本文档是「实时渲染系统」系列文档的第 **6** 篇，详细阐述声明式 FrameGraph 帧图系统的设计理念、Pass 声明规范、完整生命周期（构建/编译/执行/收尾）、资源别名复用与自动屏障推导。

---

## 1. 设计理念与动机

### 1.1 传统渲染管线的痛点

手动管理渲染 Pass 依赖与 Vulkan 同步面临以下问题：

```
痛点 1: 手动排序
  100个Pass之间有复杂读写依赖 → 人工排序极易出错

痛点 2: 手动屏障
  每个Pass前后需精确的VkImageMemoryBarrier → 数百个屏障手写极易遗漏

痛点 3: 布局转换
  纹理在不同Pass间需不同ImageLayout → 手动管理转换时机容易死锁或性能浪费

痛点 4: 资源浪费
  每个Pass独立分配RT/Buffer → 大量显存闲置浪费

痛点 5: 代码耦合
  管线创建、资源分配、同步逻辑混在一起 → 维护困难、无法并行开发
```

### 1.2 FrameGraph 的解决思路

FrameGraph 采用**声明式设计**，工程师只声明"我要什么"，系统自动推导"如何安排"：

```
声明式思路:
  Pass声明: "我需要读取GBuffer的深度和法线, 写入光照结果"
  FrameGraph自动推导:
    ├─ GBuffer Pass必须在光照Pass之前执行
    ├─ GBuffer→光照之间需 VkImageMemoryBarrier (深度: DEPTH→SHADER_READ)
    ├─ 光照Pass之后深度不再被读取 → 可释放/别名复用
    └─ 执行顺序: GBuffer → Barrier → 光照 → ...
```

### 1.3 核心抽象：虚拟资源句柄

**FrameGraph 内部所有资源使用虚拟句柄引用，不硬编码 VkImage/VkBuffer**：

```cpp
// 虚拟资源句柄 (编译前使用)
struct VirtualResource {
    ResourceID id;
    ResourceType type;     // IMAGE / BUFFER
    Format format;
    Extent3D extent;
    bool persistent;       // 跨帧持久 vs 单帧临时
};

// 物理资源 (编译后映射)
struct PhysicalResource {
    VkImage image;         // 或 VkBuffer
    VmaAllocation allocation;
    VkImageView view;
    uint32_t bindlessIndex;
};

// 编译阶段: VirtualResource → PhysicalResource 映射
std::unordered_map<ResourceID, PhysicalResource> resourceMapping;
```

---

## 2. Pass 声明规范

每个 Pass 是帧图的最小单元，声明五组信息：

### 2.1 五组声明信息

```
┌──────────────────────────────────────────────────────────────┐
│  Pass 声明规范                                                 │
│                                                              │
│  1. 绑定管线类型 + Shader入口                                  │
│     └─ 管线类型: 图形/计算/光追                                │
│     └─ Shader: Compute "LightingCull" / Graphics "PBR.vert+frag" │
│                                                              │
│  2. 资源布局                                                   │
│     └─ 全局Bindless表 (Set 0: 纹理/Buffer/AS数组)              │
│     └─ 帧Uniform (Set 1: 相机/光照/时间)                       │
│     └─ 内部临时资源 (Set 2: Pass级输入/输出)                    │
│                                                              │
│  3. 附件信息                                                   │
│     └─ 色彩附件: 格式 + Load/Store操作 + 清除参数              │
│     └─ 深度/模板附件: 格式 + Load/Store操作 + 清除深度          │
│     └─ 输入附件: Subpass InputAttachment (移动端)              │
│                                                              │
│  4. 依赖关系                                                   │
│     └─ 读取哪些Pass的输出资源                                  │
│     └─ 写入哪些资源 (供后续Pass消费)                           │
│                                                              │
│  5. 自动生成                                                   │
│     └─ Shader头文件 (从反射自动生成绑定代码)                    │
│     └─ PipelineLayout (描述符集布局+Push Constant)             │
│     └─ Bindless资源索引 (全局纹理/Buffer索引)                   │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 Pass 声明代码示例

```cpp
// Pass声明: GBuffer生成
class GBufferPass : public FrameGraphPass {
public:
    // 1. 绑定管线类型 + Shader
    PipelineType GetPipelineType() const override {
        return PipelineType::GRAPHICS_MESH;  // Mesh Shader图形管线
    }
    const char* GetShaderPath() const override {
        return "gbuffer_mesh";  // Mesh + Fragment Shader
    }

    // 2. 资源布局
    void DeclareResources(ResourceBuilder& builder) override {
        builder.UseGlobalBindlessSet(SET_0);       // 全局纹理/Buffer数组
        builder.UseFrameUniformSet(SET_1);          // 帧Uniform: 相机/时间
        builder.DeclareInternalResource("GBufferAlbedo",
            ResourceType::IMAGE, VK_FORMAT_R8G8B8A8_UNORM, renderExtent);
        builder.DeclareInternalResource("GBufferNormal",
            ResourceType::IMAGE, VK_FORMAT_R8G8B8A8_UNORM, renderExtent);
        builder.DeclareInternalResource("GBufferORM",
            ResourceType::IMAGE, VK8G8B8A8_UNORM, renderExtent);
        builder.DeclareInternalResource("GBufferMotion",
            ResourceType::IMAGE, VK_FORMAT_R16G16_SFLOAT, renderExtent);
        builder.DeclareInternalResource("DepthPrepass",
            ResourceType::IMAGE, VK_FORMAT_D32_SFLOAT, renderExtent);
    }

    // 3. 附件信息
    void DeclareAttachments(AttachmentBuilder& builder) override {
        builder.AddColorAttachment("GBufferAlbedo",
            LoadOp::CLEAR, StoreOp::STORE,
            ClearValue{0.0f, 0.0f, 0.0f, 0.0f});
        builder.AddColorAttachment("GBufferNormal",
            LoadOp::CLEAR, StoreOp::STORE,
            ClearValue{0.0f, 0.0f, 0.0f, 0.0f});
        builder.AddColorAttachment("GBufferORM",
            LoadOp::CLEAR, StoreOp::STORE,
            ClearValue{0.0f, 0.0f, 0.0f, 0.0f});
        builder.AddColorAttachment("GBufferMotion",
            LoadOp::CLEAR, StoreOp::STORE,
            ClearValue{0.0f, 0.0f, 0.0f, 0.0f});
        builder.AddDepthStencilAttachment("DepthPrepass",
            LoadOp::CLEAR, StoreOp::STORE,
            ClearValue{1.0f, 0});
    }

    // 4. 依赖关系
    void DeclareDependencies(DependencyBuilder& builder) override {
        builder.ReadResource("HiZDepthPyramid");   // 读取预处理阶段的深度金字塔
        builder.ReadResource("IndirectDrawCommands");  // 读取GPU剔除后的间接命令
        builder.WriteResource("GBufferAlbedo");    // 写入GBuffer (供光照Pass读取)
        builder.WriteResource("GBufferNormal");
        builder.WriteResource("GBufferORM");
        builder.WriteResource("GBufferMotion");
        builder.WriteResource("DepthPrepass");
    }

    // 5. 执行回调
    void Execute(const PassExecutionContext& ctx) override {
        // 编译后, ctx包含映射好的物理资源和管线
        vkCmdDrawMeshTasksIndirectEXT(ctx.cmdBuffer,
            ctx.GetBuffer("IndirectDrawCommands"), 0, 1, 0);
    }
};
```

### 2.3 Pass 注册机制

```cpp
// 引擎初始化时注册所有标准Pass模板
class FrameGraphRegistry {
    std::unordered_map<std::string, PassTemplate> templates;

public:
    void RegisterStandardPasses() {
        Register("GBuffer",     GBufferPass::GetTemplate());
        Register("ShadowMap",   ShadowMapPass::GetTemplate());
        Register("Lighting",    LightingPass::GetTemplate());
        Register("SSR",         SSRPass::GetTemplate());
        Register("TAA",         TAAPass::GetTemplate());
        Register("Bloom",       BloomPass::GetTemplate());
        Register("ToneMapping", ToneMappingPass::GetTemplate());
        Register("UI",          UIPass::GetTemplate());
        // ... 根据架构选型注册对应Pass集合
    }
};
```

---

## 3. 帧图完整生命周期

FrameGraph 每帧动态构建，经历四个阶段：

```
┌─────────────────────────────────────────────────────┐
│  帧图生命周期                                         │
│                                                      │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐ │
│  │  构建     │──▶│  编译     │──▶│  执行     │──▶│  收尾     │ │
│  │  (收集Pass) │  │  (优化DAG) │  │  (GPU执行) │  │  (回收)   │ │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘ │
└─────────────────────────────────────────────────────┘
```

### 3.1 构建阶段

**收集当前帧所有活跃 Pass**，根据场景开关和可见性剔除无效节点：

```cpp
void FrameGraph::Build(const SceneState& scene) {
    // 根据渲染架构和场景配置决定活跃Pass
    if (scene.hasOpaqueObjects)     AddPass("GBuffer");
    if (scene.lightCount > 0)      AddPass("ShadowMap");
    if (scene.hasSSR)              AddPass("SSR");
    if (scene.hasRTReflections && deviceSupportsRT)
                                   AddPass("RTReflection");
    if (scene.hasTransparentObjects) AddPass("Transparent");
    if (scene.hasSky)              AddPass("Sky");
    AddPass("Lighting");           // 基础光照始终活跃
    AddPass("TAA");                // TAA始终活跃
    if (scene.hasBloom)            AddPass("Bloom");
    AddPass("ToneMapping");        // 基础色调映射始终活跃
    if (scene.hasDOF)              AddPass("DOF");
    AddPass("ColorGrading");       // 色彩调色始终活跃
    AddPass("UI");                 // UI始终活跃

    // 构建DAG有向无环图
    for (auto& pass : activePasses) {
        pass.DeclareDependencies(dependencyBuilder);
    }
    BuildDAG();  // 根据依赖关系构建边
}
```

**DAG构建示例**：

```
活跃Pass DAG (当前帧配置: 标准延迟渲染 + SSR + TAA):

  HiZDepth ──▶ GBuffer ──┬──▶ ShadowMap ──▶ Lighting ──▶ TAA ──▶ ToneMapping ──▶ ...
                         │                  │
                         └─▶ SSR ───────────┘ (SSR依赖GBuffer+光照)
                         │
                         └─▶ Transparent ──▶ (后处理链)

不活跃Pass (被剔除):
  RTReflection (场景未启用光追反射)
  DOF (场景未启用景深)
```

### 3.2 编译阶段（核心优化流程）

编译阶段是 FrameGraph 的**核心智能环节**，执行五项优化：

#### 优化 1: 无效 Pass 剔除 + 依赖环路校验

```cpp
void FrameGraph::Compile() {
    // 1a. 无效Pass剔除: 输出不被任何Pass消费的Pass可剔除
    // 例如: 如果TAA被禁用, 则运动矢量Pass的输出不被消费 → 可剔除
    RemoveUnusedPasses();

    // 1b. 环路校验: DAG不允许环路
    if (DetectCycle()) {
        LogError("FrameGraph dependency cycle detected!");
        // 自动断开环路: 将环路中最弱依赖边移除
    }
}
```

#### 优化 2: 资源生命周期计算 + 显存别名复用

**核心优化**：不同 Pass 分时共用同一块 RT/Buffer 的物理显存：

```
无别名复用:
  GBuffer(100MB) + LightingOutput(50MB) + BloomTemp(30MB) = 180MB显存

有别名复用 (GBuffer在光照后不再被读取):
  GBuffer(100MB) → 光照完成后释放 → BloomTemp复用GBuffer显存(30MB < 100MB)
  LightingOutput(50MB) → TAA完成后释放 → ToneMappingTemp复用(不需要额外显存)
  实际显存: 100MB + 50MB = 150MB (节省30MB)
```

```cpp
void FrameGraph::ComputeResourceLifetimes() {
    // 计算每个资源的首次写入Pass和最后一次读取Pass
    for (auto& resource : declaredResources) {
        resource.firstWritePass = FindFirstWriter(resource.id);
        resource.lastReadPass   = FindLastReader(resource.id);
        resource.lifetime = {firstWritePass, lastReadPass};
    }

    // 别名复用: 不同生命周期(无重叠)的资源共享物理显存
    AliasOptimizer optimizer;
    for (auto& resA : declaredResources) {
        for (auto& resB : declaredResources) {
            if (resA.lifetime.end < resB.lifetime.start) {
                // resA的生命周期在resB之前结束 → 可复用
                if (resA.physicalSize >= resB.physicalSize) {
                    optimizer.AddAlias(resA.id, resB.id);  // resB使用resA的物理显存
                }
            }
        }
    }
}
```

#### 优化 3: 自动推导内存屏障与布局转换

FrameGraph 编译器根据 Pass 间资源读写依赖，自动生成 Vulkan 内存屏障：

```cpp
void FrameGraph::GenerateBarriers() {
    for (size_t i = 1; i < executionOrder.size(); ++i) {
        auto& prevPass = executionOrder[i-1];
        auto& currPass = executionOrder[i];

        // 检查从prevPass到currPass之间需要屏障的资源
        for (auto& resource : prevPass.writtenResources) {
            if (currPass.readsResource(resource.id) ||
                currPass.writesResource(resource.id)) {
                // 生成屏障
                VkImageMemoryBarrier barrier = GenerateBarrier(resource, prevPass, currPass);
                barriers.push_back(barbar);
            }
        }
    }
}

VkImageMemoryBarrier FrameGraph::GenerateBarrier(
    const Resource& resource, const Pass& writer, const Pass& reader) {

    VkImageMemoryBarrier barrier = {};
    barrier.sType = VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER;

    // 自动推导旧/新布局
    barrier.oldLayout = GetLayoutForPass(resource, writer);   // 如 COLOR_ATTACHMENT_OPTIMAL
    barrier.newLayout = GetLayoutForPass(resource, reader);   // 如 SHADER_READ_ONLY_OPTIMAL

    // 自动推导源/目标访问标志
    barrier.srcAccessMask = GetAccessMaskForPass(resource, writer);  // 如 COLOR_ATTACHMENT_WRITE
    barrier.dstAccessMask = GetAccessMaskForPass(resource, reader);  // 如 SHADER_READ

    // 自动推导源/目标管线阶段
    barrier.srcStageMask = GetStageMaskForPass(writer);  // 如 COLOR_ATTACHMENT_OUTPUT
    barrier.dstStageMask = GetStageMaskForPass(reader);  // 如 FRAGMENT_SHADER

    barrier.image = resource.physicalImage;
    barrier.subresourceRange = {VK_IMAGE_ASPECT_COLOR_BIT, 0, resource.mipLevels, 0, 1};

    return barrier;
}
```

**屏障推导规则表**：

| 写入Pass | 写入布局 | 写入Stage | 写入Access | → 读取Pass | 读取布局 | 读取Stage | 读取Access |
|---------|---------|----------|-----------|----------|---------|----------|-----------|
| GBuffer | COLOR_ATTACHMENT | COLOR_ATTACHMENT_OUTPUT | COLOR_ATTACHMENT_WRITE | Lighting | SHADER_READ_ONLY | FRAGMENT_SHADER | SHADER_READ |
| Depth预通道 | DEPTH_ATTACHMENT | EARLY_FRAGMENT_TESTS | DEPTH_WRITE | GBuffer | DEPTH_READ_ONLY | FRAGMENT_SHADER | SHADER_READ |
| ShadowMap | DEPTH_ATTACHMENT | LATE_FRAGMENT_TESTS | DEPTH_WRITE | Lighting | SHADER_READ_ONLY | FRAGMENT_SHADER | SHADER_READ |
| Compute输出 | GENERAL | COMPUTE_SHADER | SHADER_WRITE | Graphics | SHADER_READ_ONLY | FRAGMENT_SHADER | SHADER_READ |

#### 优化 4: 兼容 Subpass 合并（移动端 TBDR）

```cpp
void FrameGraph::MergeCompatibleSubpasses() {
    // 检查连续Pass是否可合并为同一RenderPass的Subpass
    for (size_t i = 0; i < executionOrder.size(); ++i) {
        auto& passA = executionOrder[i];
        for (size_t j = i+1; j < executionOrder.size(); ++j) {
            auto& passB = executionOrder[j];

            if (CanMergeAsSubpass(passA, passB)) {
                // 合并条件:
                // 1. 两者都是图形管线
                // 2. passB仅读取passA的输出(InputAttachment)
                // 3. 两者使用相同帧缓冲附件集(部分附件可复用)
                // 4. passB不读取其他Pass的输出(仅依赖passA)

                MergeIntoSubpass(passA, passB);
                // 收益: 移动端TBDR片上存储避免显存读写
            }
        }
    }
}
```

**经典合并案例**：GBuffer + Lighting 合并为 Subpass（TDBR 方案）

#### 优化 5: 管线缓存查找 / 动态创建

```cpp
void FrameGraph::ResolvePipelines() {
    for (auto& pass : executionOrder) {
        // 查找管线缓存
        PipelineKey key = ComputePipelineKey(pass);

        if (auto pipeline = pipelineCache.Find(key)) {
            pass.pipeline = pipeline;  // 命中缓存
        } else {
            // 按需创建新管线
            pass.pipeline = pipelineCache.Create(key, pass);
        }
    }
}
```

### 3.3 执行阶段

```cpp
void FrameGraph::Execute() {
    // 多线程并行录制CommandBuffer
    // 按拓扑顺序提交队列

    // 1. 按执行计划逐Pass录制
    for (auto& pass : executionOrder) {
        auto& ctx = passContexts[pass.index];

        // 插入自动生成的屏障
        for (auto& barrier : pass.preBarriers) {
            vkCmdPipelineBarrier(cmdBuffer,
                barrier.srcStageMask, barrier.dstStageMask,
                0, 0, nullptr, 0, nullptr, 1, &barrier);
        }

        // 绑定管线
        if (pass.pipelineType == PipelineType::COMPUTE) {
            vkCmdBindPipeline(cmdBuffer, VK_PIPELINE_BIND_POINT_COMPUTE, pass.pipeline);
        } else if (pass.pipelineType == PipelineType::GRAPHICS) {
            vkCmdBindPipeline(cmdBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, pass.pipeline);
        }

        // 绑定描述符集
        vkCmdBindDescriptorSets(cmdBuffer, pass.pipelineType, pass.layout,
            0, pass.descriptorSetCount, pass.descriptorSets, 0, nullptr);

        // 执行Pass回调
        pass.Execute(ctx);

        // 插入Pass后屏障(如有)
        for (auto& barrier : pass.postBarriers) {
            vkCmdPipelineBarrier(cmdBuffer, ...);
        }
    }

    // 2. 配合Timeline信号量+Fence做队列同步
    // 多队列并行: 计算队列与图形队列并行执行
    SubmitToQueues();
}
```

**多线程录制**：

```cpp
// 按Pass分配到不同线程录制
void FrameGraph::ParallelRecord() {
    // 将Pass分组为可并行录制的批次
    // 同一批次内的Pass之间无直接依赖
    auto batches = ComputeParallelBatches(executionOrder);

    for (auto& batch : batches) {
        // 并行录制各Pass的CommandBuffer
        std::vector<std::future<VkCommandBuffer>> futures;
        for (auto& pass : batch) {
            futures.push_back(std::async([&]() {
                VkCommandBuffer cmd = AllocateCommandBuffer();
                RecordPass(cmd, pass);
                return cmd;
            }));
        }

        // 收集所有CommandBuffer并提交
        for (auto& f : futures) {
            auto cmd = f.get();
            SubmitCommandBuffer(cmd);
        }
    }
}
```

**多队列并行执行**：

```
计算队列 (Compute Queue):
  ──▶ GPU剔除 ──▶ HiZ生成 ──▶ SSR ──▶ Bloom降采样
  Timeline Semaphore: signal at each completion point

图形队列 (Graphics Queue):
  ──▶ GBuffer ──▶ ShadowMap ──▶ Lighting ──▶ 透明 ──▶ 后处理链
  Wait on Compute Queue's Timeline Semaphore at dependency points

异步传输队列 (Transfer Queue):
  ──▶ 资源上传 ──▶ BLAS构建
  Signal Fence after completion
```

### 3.4 收尾阶段

```cpp
void FrameGraph::FinishFrame() {
    // 1. 延迟释放过期资源
    deferredReleaseQueue.ProcessCompleted(lastCompletedFrame);

    // 2. 重置环形缓冲
    ringUniformBuffer.RecycleCompletedSegments();

    // 3. 重置帧级描述符池
    vkResetDescriptorPool(device, frameDescriptorPool, 0);

    // 4. 更新历史帧资源（TAA历史帧等跨帧持久资源）
    SwapHistoryFrames();

    // 5. 回收别名复用的物理显存
    // (别名复用的资源在最后一次读取Pass完成后即可被下一个复用者接管)

    // 6. 更新帧计数器
    currentFrameIndex++;
}
```

---

## 4. FrameGraph 编译优化效果量化

### 4.1 典型帧图优化效果

| 优化项 | 无优化 | 有优化 | 节省 |
|--------|--------|--------|------|
| **活跃Pass数** | 全部Pass(25个) | 动态剔除(15个) | 40%GPU执行时间 |
| **显存占用** | 每Pass独立分配(300MB) | 别名复用(180MB) | 40%显存 |
| **屏障数量** | 手写保守屏障(50个) | 精确推导(15个) | 70%管线阻塞 |
| **Subpass合并** | 无合并(GBuffer→显存→Lighting→显存) | 合并(GBuffer→片上→Lighting→显存) | 50%带宽(移动端) |
| **管线创建** | 每帧按需创建(慢) | PipelineCache命中(快) | 80%管线创建时间 |

### 4.2 无效Pass剔除案例

```
场景: 低画质档位, 禁用SSR/Bloom/DOF/光追

全Pass列表 (25个):
  HiZ, GPU-Cull, GBuffer, ShadowMap, Lighting, SSR, RT-Shadow,
  RT-Reflection, Transparent, Sky, Particles, MotionVector, TAA,
  FSR, ToneMapping, Bloom, DOF, MotionBlur, ColorGrading, UI, ...

动态剔除后 (12个):
  HiZ, GPU-Cull, GBuffer, ShadowMap, Lighting, Transparent, Sky,
  TAA, ToneMapping, ColorGrading, UI

  剔除: SSR(禁用), RT-Shadow(无RT), RT-Reflection(无RT),
        Bloom(禁用), DOF(禁用), MotionBlur(禁用),
        FSR(低画质不需要超分), Particles(场景无粒子),
        MotionVector(TAA低画质模式不依赖运动矢量)
```

---

## 5. FrameGraph 与 Vulkan API 的映射

| FrameGraph 概念 | Vulkan 对象 | 生成时机 |
|----------------|-------------|---------|
| VirtualResource → PhysicalResource | VkImage/VkBuffer + VmaAllocation | 编译阶段 |
| Pass → Pipeline | VkPipeline + VkPipelineLayout | 编译阶段(PipelineCache查找/创建) |
| Pass → DescriptorSet | VkDescriptorSet | 编译阶段(描述符池分配) |
| 依赖 → Barrier | VkImageMemoryBarrier/VkBufferMemoryBarrier | 编译阶段(自动推导) |
| 依赖 → Layout Transition | VkImageLayout 转换 | 编译阶段(自动推导) |
| Subpass合并 → RenderPass | VkRenderPass + VkFramebuffer | 编译阶段(Subpass优化) |
| 执行顺序 → CommandBuffer | VkCommandBuffer 录制 | 执行阶段(多线程并行) |
| 多队列 → Timeline Semaphore | VkSemaphore (Timeline) | 执行阶段(队列同步) |

---

*上一篇：[05-资源管理体系](05-资源管理体系.md)*
*下一篇：[07-RDG与FrameGraph概念辨析](07-RDG与FrameGraph概念辨析.md)*
