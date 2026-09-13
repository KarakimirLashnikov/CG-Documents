# 移动端 GPU 架构 — 移动端延迟渲染与 Subpass 实践

> 本文档是「移动端 GPU 架构」系列文档的第 **4** 篇。本篇讨论一个被广泛误传的命题：「延迟渲染是移动端的天作之合」。我们会给出收益与代价的完整账本、Forward / Forward+ / Deferred 的决策依据，以及 Vulkan 上「片上闭环」的两条正确实现路径。

---

## 1. 命题审查：延迟渲染在移动端到底如何

### 1.1 通行说法 vs 事实

| 通行说法 | 事实 |
|---------|------|
| 「延迟渲染在桌面端吃带宽，在移动端 TBR 下成了天作之合」 | 🟠 TBDR 把 deferred 的**最大缺点从「致命」降为「可接受」**，所以 deferred **可行**。但不等于「最优」，更不等于「天作之合」 |
| 「延迟渲染完美契合 Subpass 机制，是移动端标配」 | 🟠 契合是真的（Khronos 承认 subpass 设计受 tile-based 影响），但 UE 移动端**默认仍是 Forward** |
| 「延迟渲染解决了移动端小三角形痛点」 | ❌ 与分箱无关。分箱成本由**图元数量与 VS 复杂度**决定，deferred 不减少图元 |

### 1.2 两条官方立场（方向相反，必须都看）

#### 立场 A：Epic / UE 对移动端 Deferred 的评估（倾向支持）

UE 官方《Mobile Rendering and Shading Modes》要点：

| 维度 | Mobile Forward（**默认**） | Mobile Deferred |
|------|--------------------------|----------------|
| 光照处理时机 | 绘制时与材质一起算 | 几何 / 光照分离为两个阶段 |
| 基线性能 | 最快，兼容性最广 | 需要较高端硬件 |
| 抗锯齿选项 | **最好**（支持 MSAA） | **受限**（无法使用 MSAA） |
| 预计算光照 | ✅ 更快，且支持全部 shading model | ❌ 只支持 DefaultLit 和 Unlit |
| 动态光照 / 阴影 / 反射 | 支持有限 | ✅ 支持 Light Function / IES / Lit Decal 等 |
| 材质复杂度 | 高（每个材质都要包含光照与阴影代码 → 指令数、采样器数、编译时间都上升） | 低（示例：同一材质 Forward 147 指令 / 2 采样器 → Deferred 34 指令 / 0 采样器） |
| 何时优选 | 用预计算光照；用局部光源；大量反射捕获；目标设备不是 tile-based GPU | 用动态光照；大量室外场景；需要高级光照特性 |

**UE 的结论**：

> - 对于使用预计算光照的项目，**强烈推荐 Forward**
> - 对大多数移动项目，Deferred 的性能收益显著，**推荐使用 Mobile Deferred**

#### 立场 B：Meta / Oculus 对移动 VR 的评估（倾向劝退）

Oculus 官方《PC Rendering Techniques to Avoid when Developing for Mobile VR》明确把 **Deferred Rendering 和 Depth Pre-pass** 列为「不要做」的两项：

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  Meta 对移动 VR 用 Deferred 的量化反对理由                                     │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  核心概念：Resolve Cost（解析成本）                                            │
│    每个被渲染到的贴图都需要一次 Resolve（片上 → 主存）                           │
│    Quest 1 上实测：每只眼的 eye buffer 约 0.5 ms                               │
│                                                                              │
│  Forward 管线：  ████ ~1 ms                                                  │
│  Deferred 管线： ████████████ ~3 ms+   ← 多出的 2ms 全是 Resolve              │
│                                                                              │
│  在 60fps（16.67ms 预算）的 VR 场景里，多花 2ms 是不可接受的                     │
│                                                                              │
│  附加理由：                                                                    │
│    · Deferred 只在「几何复杂 + 光源多」时才有优势                                │
│      而移动端 GPU 既推不了大量顶点，也做不了大量像素填充                           │
│    · 透明物体无法走 deferred，仍要 forward pass                                │
└──────────────────────────────────────────────────────────────────────────────┘
```

Oculus 对 Depth Pre-pass 的反对理由（同样适用于移动端）：

| 理由 | 说明 |
|------|------|
| 排序良好时收益很小 | 前后排序后普通深度测试就能剔除，pre-pass 只对排序差或互相遮挡的几何有用 |
| draw call 翻倍 | 所有几何要提交两次；移动端 draw call 是 CPU 侧的重负担 |
| 顶点处理翻倍 | 移动端顶点处理相对更贵，多处理一遍顶点通常比省下的像素填充更亏 |

### 1.3 为什么两者结论不同

关键差异在于**场景特征**：

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  UE 移动端场景（第三人称手游 / 大世界）  vs  移动 VR（Quest）                    │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  维度              UE 移动端手游            Quest 移动 VR                      │
│  ─────────────────────────────────────────────────────────────────────────── │
│  分辨率            1080p 单屏，可 720p 渲染   双目，每眼 ~1600×1760，合计更高     │
│  帧率目标          30–60 fps                 72–90 fps（硬性要求）              │
│  帧预算            16.7–33 ms                11–13.9 ms                       │
│  光照             大量动态光（大世界）        以烘焙为主，动态光有限               │
│  延迟敏感度        中等                      极高（延迟抖动会导致晕动）           │
│  Resolve 相对代价  较低（单屏、预算宽）        极高（双目、预算紧）                │
│                                                                              │
│  → 结论：Deferred 的收益（降低材质复杂度、支持多动态光）在 UE 场景中显著            │
│          Deferred 的代价（Resolve）在 VR 场景中被放大到不可接受                   │
└──────────────────────────────────────────────────────────────────────────────┘
```

**所以正确的表述是**：

> 移动端 Deferred 是一个**有明确适用边界的选项**，而不是默认最优解。判断依据是：动态光源数量、材质复杂度、Resolve 预算，以及是否需要 MSAA 与预计算光照。

---

## 2. 决策：Forward / Forward+ / Deferred

### 2.1 三种管线的成本结构对比

| 维度 | Forward | Forward+ / Clustered | Deferred |
|------|---------|---------------------|----------|
| G-Buffer | ❌ 无 | ❌ 无 | ✅ 有（4–20 字节/像素） |
| 光照数据结构 | 无 | 光源簇（Clustered Light Grid），几十 KB–几 MB | 无（或存光列表在 tile 上） |
| 每像素光照成本 | 与光源数线性相关（或受 uniform 分支限制） | 与**该簇内**光源数相关 | 与**该像素实际受光**数相关 |
| 每 draw 的光照指令 | 每个材质都要编译进光照代码 | 同 Forward（但光照数可控） | 完全没有（材质 shader 极简） |
| Shader 排列组合 | **爆炸**（材质 × 光照配置） | 中等 | **极少** |
| 编译时间 | 长 | 中等 | 短 |
| MSAA | ✅ 完整支持 | ✅ 支持 | ❌ **不可用**（语义冲突） |
| 透明物体 | ✅ 天然支持 | ✅ 天然支持 | ❌ 必须另开 forward pass |
| 显存 / 片上占用 | 低 | 中（光源簇数据） | **高**（G-Buffer MRT） |
| 带宽 | 低 | 低 | 中（若片上闭环）/ 高（若落主存） |
| CPU 侧（RHI 线程） | 高（state 绑定多） | 中 | **低**（state 少） |
| 移动端适配度 | ✅✅ 最稳 | ✅ 推荐 | ⚠️ 有条件可行 |

### 2.2 决策树

```
                        开始
                          │
                          ▼
        ┌─────────────────────────────────────┐
        │ 项目是否主要使用预计算（烘焙）光照？      │
        └──────┬───────────────────────┬──────┘
               │ 是                     │ 否
               ▼                        ▼
        ┌─────────────┐      ┌──────────────────────────────┐
        │ 用 Forward   │      │ 动态光源数量是否很多（>50）？    │
        │（UE 官方推荐）│      └──────┬──────────────┬────────┘
        └─────────────┘             │ 是            │ 否
                                    ▼                ▼
                      ┌──────────────────────┐  ┌──────────────────┐
                      │ 是否需要 MSAA？        │  │ 用 Forward+ 即可  │
                      └───┬──────────┬───────┘  │（成本更低）        │
                          │ 是        │ 否      └──────────────────┘
                          ▼           ▼
                ┌──────────────┐  ┌──────────────────────────────┐
                │ Forward+     │  │ 用 Deferred                   │
                │（Clustered） │  │ · G-Buffer 极限精简（≤16 B/px）│
                └──────────────┘  │ · 单 render pass + subpass     │
                                  │ · 无 MSAA，改用 FXAA/TAA       │
                                  │ · 透明单独 forward pass        │
                                  └──────────────────────────────┘

        ⚠️ 额外约束：若目标是移动 VR（高帧率 + 双目）
            → Resolve 预算极紧，Deferred 需要非常谨慎的评估
            → 优先 Forward / Forward+
```

### 2.3 混合管线：移动端的现实选择

大量移动端项目实际采用**混合方案**：

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  移动端混合管线（示例）                                                        │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ① 不透明几何 → Deferred G-Buffer（同一 render pass / subpass）               │
│       ↓ local read                                                          │
│  ② 光照 Pass（Cliustered 光源剔除 + IBL + 阴影图）                            │
│       ↓ 写颜色附件                                                           │
│  ③ 透明物体 → Forward Pass（新增 render pass，用 G-Buffer 深度做测试）         │
│       ↓                                                                     │
│  ④ 后处理（TAA/FXAA → Bloom → 色调映射 → 调色）                                │
│                                                                              │
│  ⚠️ 移动端注意：                                                              │
│     · ③ 是新 render pass，会产生一次 Resolve + 一次深度 LOAD                   │
│        → 若透明物体不多，考虑把它合并进 ② 之后的同一 pass（forward 材质分支）    │
│     · ④ 的级数能省就省。Bloom 用低分辨率 + 少量 mip，SSAO 考虑省略             │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 移动端 Deferred 的 G-Buffer 设计

### 3.1 从需求反推，而不是照搬桌面

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  G-Buffer 里每多一个字节，都要付出三重代价                                      │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ① 片上内存占用 ↑  → bin 变小 → bin 数量 ↑ → 分箱元数据 ↑、Resolve 更碎        │
│  ② 片上带宽 ↑     → 片元着色阶段的读写压力 ↑                                  │
│  ③ 若溢出         → 部分落主存 → 主存带宽 ↑、延迟 ↑                            │
│                                                                              │
│  → 反过来：每省一个字节，三重收益叠加，是移动端性价比最高的优化之一                 │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 逐字段审查表

| 字段 | 是否必须存 | 替代方案 | 建议格式 |
|------|-----------|---------|---------|
| Albedo (RGB) | ✅ 必须 | — | `RGBA8`（或与粗糙度共用 alpha） |
| Normal | ✅ 必须 | 用 Octahedron / 球面坐标编码压到 2 通道 | `RG16F` 或 `RGB10A2` |
| Roughness | ✅ 必须 | 打包进 Albedo 的 A 通道 | `RGBA8.a` |
| Metallic | ✅ 必须 | 打包进法线所在 RT 的剩余通道 | 位宽 8 bit 足够 |
| **World Position** | ❌ **不要存** | **由深度反投影重建** | — |
| Depth | ✅ 必须 | 渲染目标本身，不算 MRT | `D32F` 或 `D24S8` |
| Emissive | ⚠️ 可选 | 用单独的 forward pass 处理自发光物体 | `RGB10A2` |
| Specular / F0 | ⚠️ 可选 | 由 metallic 推导 | — |
| AO | ⚠️ 可选 | 打包进某 RT 的剩余通道 | 8 bit |
| Motion Vector | ⚠️ 取决于 TAA | 若用 TAA 必须存 | `RG16F` |
| 材质 ID | ⚠️ 取决于是否需要 | 若为 Stencil + 分支可用 stencil 位 | 8 bit |

### 3.3 三档方案对照

| 方案 | RT0 | RT1 | RT2 | Depth | 每像素合计 |
|------|-----|-----|-----|-------|-----------|
| **极简** | `RGBA8` albedo + roughness | `RGBA8` oct-normal + metallic + AO | — | `D32F` | **12 B** |
| **标准** | `RGB10A2` albedo + roughness | `RG16F` oct-normal | `RGBA8` metallic + AO + matID | `D32F` | **20 B** |
| ❌ **桌面惯性** | `RGBA16F` albedo | `RGBA16F` normal | `RGBA16F` worldPos | `D24S8` | **28 B+** |

**关于 Octahedron 编码**：把单位球面法线映射到 \([-1,1]^2\) 的平面（八面体映射）。用 `RG16F` 存两个 fp16，精度对移动端视觉质量足够；若担心精度，可用 `RGB10A2` 存两个 10-bit 分量。

```glsl
// 八面体编码（几何 Pass 写 G-Buffer）
vec2 octWrap(vec2 v) {
    return (1.0 - abs(v.yx)) * vec2(v.x >= 0.0 ? 1.0 : -1.0,
                                    v.y >= 0.0 ? 1.0 : -1.0);
}
vec2 encodeNormalOct(vec3 n) {
    n /= (abs(n.x) + abs(n.y) + abs(n.z));
    n.xy = n.z >= 0.0 ? n.xy : octWrap(n.xy);
    return n.xy * 0.5 + 0.5;   // → [0,1] 便于存 RG16F 或 UNORM
}

// 解码（光照 Pass 读 G-Buffer）
vec3 decodeNormalOct(vec2 e) {
    e = e * 2.0 - 1.0;
    vec3 n = vec3(e.xy, 1.0 - abs(e.x) - abs(e.y));
    float t = max(-n.z, 0.0);
    n.xy += vec2(n.x >= 0.0 ? -t : t, n.y >= 0.0 ? -t : t);
    return normalize(n);
}
```

```glsl
// 从深度重建世界位置（不存 worldPos 的代价：光照 Pass 里多算一次矩阵逆变换）
vec3 reconstructWorldPos(vec2 uv, float depth, mat4 invViewProj) {
    vec4 clip = vec4(uv * 2.0 - 1.0, depth, 1.0);
    vec4 world = invViewProj * clip;
    return world.xyz / world.w;
}
```

⚠️ **注意**：`D32F` 与 `D24S8` 的选择取决于是否需要模板。若需要模板做材质分支，用 `D24S8`；否则 `D32F` 精度更好（深度重建世界位置的误差更小）。

### 3.4 MSAA：为什么在 G-Buffer 上不可用

⚠️ 资料给出的理由是「GMEM 尺寸 ×4 撑爆」，这只是次要原因。**主要原因是语义冲突**：

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  MSAA 的工作模型 vs Deferred 的工作模型                                        │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  MSAA 的设计前提：                                                            │
│    · 几何覆盖（coverage）逐 sample 记录                                       │
│    · **着色在 resolve 之后按像素执行一次**（shade once per pixel）             │
│    · resolve 时按 coverage 加权平均得到最终像素颜色                             │
│                                                                              │
│  Deferred 的前提：                                                            │
│    · G-Buffer 存的是**着色输入**（albedo / normal / roughness）              │
│    · 光照是非线性的（normalize、pow、specular）                               │
│                                                                              │
│  → 如果 G-Buffer 是 MSAA 的，resolve 会**平均掉不同 sample 的法线与材质**        │
│    然后在错误（被平均过）的输入上做着色                                       │
│  → 结果：法线被抹平、边缘出现错误的材质混色                                     │
│                                                                              │
│  所谓"deferred 无法 MSAA"，本质是：                                            │
│    非线性着色不能对输入做线性平均。                                              │
│    UE 官方的表述：需要"shade each sample rather than each pixel"，             │
│    光照 Pass 的成本随采样数线性上升。                                           │
└──────────────────────────────────────────────────────────────────────────────┘
```

**移动端 AA 的正确选择**：

| 方案 | 成本 | 移动端适用性 | 说明 |
|------|------|------------|------|
| **MSAA** | 中（片上 resolve 相对便宜） | ❌ Deferred 不可用；✅ Forward 可用 | Forward 管线的最佳选择；Adreno 上 2x 近乎免费 |
| **FXAA** | 极低 | ✅ Deferred 标配 | 后处理，单 pass，纯图像空间，会轻微糊化文本/UI |
| **TAA** | 中 | ⚠️ 可用但需谨慎 | 需要 Motion Vector RT（+4 B/px）与历史帧（额外带宽）；移动端分辨率低时容易出鬼影 |
| **SMAA** | 低-中 | ✅ 可用 | 比 FXAA 质量好，成本略高 |
| **超采样（SSAA）** | 高 | ❌ 移动端不用 | — |
| **MSAA（仅前向部分）** | 中 | ⚠️ 混合管线可行 | 不透明走 Deferred 无 AA + 透明走 Forward MSAA，方案复杂 |

⚠️ **TAA 在移动端的隐藏成本**：需要一张历史帧颜色缓冲（全分辨率）+ 一张 Motion Vector G-Buffer。在片上内存吃紧的平台（Mali），这 4 B/px 的 Motion Vector 可能直接把 G-Buffer 顶出片上容量。

---

## 4. 实现路径 A：Render Pass + Subpass（Vulkan 1.0–1.3）

### 4.1 为什么这段代码结构如此重要

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ✅ 正确结构：一个 render pass，两个 subpass                                   │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────┐      │
│  │ VkRenderPass (1 个 render pass instance)                            │      │
│  │  ┌──────────────────────────────────────────────────────────┐      │      │
│  │  │ Subpass 0：几何 Pass → 写 G-Buffer（color attachment）     │      │      │
│  │  └──────────────────────────┬───────────────────────────────┘      │      │
│  │                    BY_REGION 依赖                                   │      │
│  │  ┌──────────────────────────▼───────────────────────────────┐      │      │
│  │  │ Subpass 1：光照 Pass → 读 G-Buffer（input attachment）     │      │      │
│  │  │                → 写最终颜色（color attachment）             │      │      │
│  │  └──────────────────────────────────────────────────────────┘      │      │
│  └────────────────────────────────────────────────────────────────────┘      │
│                                                                              │
│  驱动解读：G-Buffer 只在这个 render pass instance 内被使用，                    │
│            且依赖是 framebuffer-local → 数据可以留在片上                        │
└──────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  ❌ 错误结构：两个独立 render pass                                            │
│                                                                              │
│  ┌───────────────────────────────┐    ┌───────────────────────────────┐     │
│  │ VkRenderPass A                 │    │ VkRenderPass B                │     │
│  │ Subpass 0：几何 → G-Buffer      │───▶│ Subpass 0：光照               │     │
│  └───────────────────────────────┘    │  采样 G-Buffer 作为纹理         │     │
│         ↑ G-Buffer 必须写回主存        └───────────────────────────────┘     │
│         ↑ 还需转到 SHADER_READ_ONLY 布局                                     │
│                                                                              │
│  驱动解读：两个 pass 之间没有任何局部关联保证                                    │
│            → G-Buffer 必须落主存，再重新加载                                    │
│            → 带宽翻倍 + 额外的 layout transition + pipeline barrier            │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 完整代码骨架（vk::raii，C++23）

```cpp
// ═════════════════════════════════════════════════════════════════════════════
// 移动端 Deferred：单 render pass + 双 subpass
// ═════════════════════════════════════════════════════════════════════════════
namespace {

constexpr uint32_t kAttachmentGBuf0  = 0;   // albedo + roughness   (RGBA8)
constexpr uint32_t kAttachmentGBuf1  = 1;   // oct-normal + metal   (RGBA8)
constexpr uint32_t kAttachmentDepth  = 2;   // D32F
constexpr uint32_t kAttachmentColor  = 3;   // 最终颜色（swapchain 或离屏）

// G-Buffer 格式：移动端极限精简，12 B/px + 深度
constexpr vk::Format kGBufferFormats[2] = {
    vk::Format::eR8G8B8A8Unorm,   // RT0: albedo.rgb + roughness
    vk::Format::eR8G8B8A8Unorm,   // RT1: oct-normal.rg + metallic + ao
};

}  // namespace

void DeferredRenderer::createRenderPass() {
    std::array<vk::AttachmentDescription, 4> attachments{};

    // ① G-Buffer 两个 MRT：LOAD 与 STORE 都用 DONT_CARE
    for (int i = 0; i < 2; ++i) {
        attachments[i]
            .setFormat(kGBufferFormats[i])
            .setSamples(vk::SampleCountFlagBits::e1)              // ★ G-Buffer 绝不用 MSAA
            .setLoadOp(vk::AttachmentLoadOp::eDontCare)           // ★ 不需要旧内容
            .setStoreOp(vk::AttachmentStoreOp::eDontCare)         // ★★ 用完即弃，不写回主存
            .setStencilLoadOp(vk::AttachmentLoadOp::eDontCare)
            .setStencilStoreOp(vk::AttachmentStoreOp::eDontCare)
            .setInitialLayout(vk::ImageLayout::eUndefined)        // ★ 内容未定义，驱动可跳过初始化
            .setFinalLayout(vk::ImageLayout::eUndefined);         // ★ 之后不再被读取
    }

    // ② 深度：Clear 后 DONT_CARE（光照 pass 之后不再需要）
    attachments[kAttachmentDepth]
        .setFormat(vk::Format::eD32Sfloat)
        .setSamples(vk::SampleCountFlagBits::e1)
        .setLoadOp(vk::AttachmentLoadOp::eClear)
        .setStoreOp(vk::AttachmentStoreOp::eDontCare)             // ★ 不写回
        .setStencilLoadOp(vk::AttachmentLoadOp::eDontCare)
        .setStencilStoreOp(vk::AttachmentStoreOp::eDontCare)
        .setInitialLayout(vk::ImageLayout::eUndefined)
        .setFinalLayout(vk::ImageLayout::eUndefined);

    // ③ 最终颜色：必须写回
    attachments[kAttachmentColor]
        .setFormat(m_swapchainFormat)
        .setSamples(vk::SampleCountFlagBits::e1)
        .setLoadOp(vk::AttachmentLoadOp::eClear)                  // ★ 明确 Clear，别留 LOAD
        .setStoreOp(vk::AttachmentStoreOp::eStore)
        .setStencilLoadOp(vk::AttachmentLoadOp::eDontCare)
        .setStencilStoreOp(vk::AttachmentStoreOp::eDontCare)
        .setInitialLayout(vk::ImageLayout::eUndefined)
        .setFinalLayout(vk::ImageLayout::ePresentSrcKHR);

    // ── Subpass 0：几何 Pass ────────────────────────────────────────────────
    std::array<vk::AttachmentReference, 2> gbufColorRefs{
        vk::AttachmentReference{kAttachmentGBuf0, vk::ImageLayout::eColorAttachmentOptimal},
        vk::AttachmentReference{kAttachmentGBuf1, vk::ImageLayout::eColorAttachmentOptimal},
    };
    vk::AttachmentReference depthRef{
        kAttachmentDepth, vk::ImageLayout::eDepthStencilAttachmentOptimal};

    vk::SubpassDescription subpassGeometry{};
    subpassGeometry
        .setPipelineBindPoint(vk::PipelineBindPoint::eGraphics)
        .setColorAttachments(gbufColorRefs)
        .setPDepthStencilAttachment(&depthRef);
    // ⚠️ Subpass 0 不写最终颜色，所以颜色附件在这里没有引用

    // ── Subpass 1：光照 Pass ───────────────────────────────────────────────
    // 关键：G-Buffer 在这里变成 input attachment
    std::array<vk::AttachmentReference, 2> gbufInputRefs{
        vk::AttachmentReference{kAttachmentGBuf0, vk::ImageLayout::eShaderReadOnlyOptimal},
        vk::AttachmentReference{kAttachmentGBuf1, vk::ImageLayout::eShaderReadOnlyOptimal},
    };
    std::array<vk::AttachmentReference, 1> finalColorRef{
        vk::AttachmentReference{kAttachmentColor, vk::ImageLayout::eColorAttachmentOptimal},
    };

    vk::SubpassDescription subpassLighting{};
    subpassLighting
        .setPipelineBindPoint(vk::PipelineBindPoint::eGraphics)
        .setInputAttachments(gbufInputRefs)   // ★ 像素局部读取
        .setColorAttachments(finalColorRef);  // 写最终颜色

    std::array<vk::SubpassDescription, 2> subpasses{subpassGeometry, subpassLighting};

    // ── ★★★ 最关键的一步：subpass 依赖 ────────────────────────────────────
    // 没有 BY_REGION，驱动会走主存往返，本章全部优化归零
    std::array<vk::SubpassDependency, 2> dependencies{};

    // 依赖 1：几何 → 光照（framebuffer-local）
    dependencies[0]
        .setSrcSubpass(0)
        .setDstSubpass(1)
        .setSrcStageMask(vk::PipelineStageFlagBits::eColorAttachmentOutput)
        .setSrcAccessMask(vk::AccessFlagBits::eColorAttachmentWrite)
        .setDstStageMask(vk::PipelineStageFlagBits::eFragmentShader)
        .setDstAccessMask(vk::AccessFlagBits::eInputAttachmentRead)
        .setDependencyFlags(vk::DependencyFlagBits::eByRegion);   // ★★★ 生命线

    // 依赖 2：外部 → 几何（保证深度 Clear 等生效）
    dependencies[1]
        .setSrcSubpass(VK_SUBPASS_EXTERNAL)
        .setDstSubpass(0)
        .setSrcStageMask(vk::PipelineStageFlagBits::eColorAttachmentOutput |
                         vk::PipelineStageFlagBits::eEarlyFragmentTests)
        .setDstStageMask(vk::PipelineStageFlagBits::eColorAttachmentOutput |
                         vk::PipelineStageFlagBits::eEarlyFragmentTests)
        .setSrcAccessMask(vk::AccessFlagBits::eNone)
        .setDstAccessMask(vk::AccessFlagBits::eColorAttachmentWrite |
                          vk::AccessFlagBits::eDepthStencilAttachmentWrite)
        .setDependencyFlags(vk::DependencyFlagBits::eByRegion);

    vk::RenderPassCreateInfo rpCI{};
    rpCI
        .setAttachments(attachments)
        .setSubpasses(subpasses)
        .setDependencies(dependencies);

    m_renderPass = vk::raii::RenderPass(m_device, rpCI);
}
```

**光照 Pass 的片元着色器**：

```glsl
// ── fragment shader: deferred lighting ──────────────────────────────────────
#version 450

// ★ 用 input attachment（subpassInput），不是 sampler2D
//   这告诉驱动：读取是像素局部的，数据可以留在片上
layout(input_attachment_index = 0, set = 0, binding = 0) uniform subpassInput gbuf0;
layout(input_attachment_index = 1, set = 0, binding = 1) uniform subpassInput gbuf1;

layout(push_constant) uniform PushConstants {
    mat4 invViewProj;
    vec4 cameraPos;
    vec4 lightParams;
} pc;

layout(location = 0) in  vec2 vUV;
layout(location = 0) out vec4 outColor;

void main() {
    // ★ subpassLoad 只能读当前像素位置，这正是片上闭环的语义
    vec4  rt0      = subpassLoad(gbuf0);
    vec4  rt1      = subpassLoad(gbuf1);

    vec3  albedo    = rt0.rgb;
    float roughness = rt0.a;
    vec3  N         = decodeNormalOct(rt1.rg);
    float metallic  = rt1.b;
    float ao        = rt1.a;

    // 从深度重建世界位置（不存 worldPos 的代价在这里）
    float depth = texelFetch(depthTexture, ivec2(gl_FragCoord.xy), 0).r;
    // ⚠️ 注意：若深度是当前 subpass 的 depth attachment，
    //    在 Mali 上读它会触发 late ZS test → 禁用 HSR！
    //    更安全的做法：在几何 Pass 里输出线性深度到一个 G-Buffer 通道，
    //    或者接受这个代价并实测。
    vec3 P = reconstructWorldPos(vUV, depth, pc.invViewProj);

    vec3 color = evaluateLighting(P, N, albedo, roughness, metallic, ao);
    outColor = vec4(color, 1.0);
}
```

⚠️ **上面那个注释是一个真实的坑**：在片元着色器里读当前 subpass 的深度附件，在 Mali 上会被识别为 `Uses late ZS test`，**禁用 early ZS 与 HSR**。移动端的稳妥做法是**把线性深度直接写进某个 G-Buffer 通道**（例如 `RG16F` 的 B 通道或复用 `RGB10A2` 的剩余位数），而不是去采样深度附件。

---

## 5. 实现路径 B：Dynamic Rendering + Local Read（Vulkan 1.4 起推荐）

### 5.1 为什么需要新路径

⚠️ **重要事实**：Vulkan 1.4 起，`VkRenderPass` 已被**标记为 deprecated**。规范原文：

> "This functionality is **deprecated by Vulkan Version 1.4**."

而 `VK_KHR_dynamic_rendering_local_read`（已被 1.4 提升为核心）补上了最后一块拼图——它让 dynamic rendering 也具备 pixel local read 能力：

> "This combination can **replace core render and subpasses**, making it possible to do local reads via input attachments with dynamic rendering."

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  Render Pass + Subpass  vs  Dynamic Rendering + Local Read                    │
├──────────────────────────┬───────────────────────────────────────────────────┤
│  Vulkan 1.0 风格          │  Dynamic Rendering（1.4 推荐）                     │
├──────────────────────────┼───────────────────────────────────────────────────┤
│ `vkCmdBeginRenderPass`   │ `vkCmdBeginRendering`                            │
│ `VkRenderPassBeginInfo`  │ `VkRenderingInfo`                                │
│ 附件由 `VkFramebuffer`    │ 附件由 `VkRenderingAttachmentInfo`                │
│ 引用                      │ 引用（栈上分配，无需 framebuffer 对象）             │
│ 管线创建引用 VkRenderPass │ 管线创建引用 `VkPipelineRenderingCreateInfo`      │
│ subpass 用 `vkCmdNextSubpass` 推进 │ 用**带 BY_REGION 的 pipeline barrier** 分隔 │
│ 布局：用 `VkImageMemoryBarrier` 转 layout | 新布局 `VK_IMAGE_LAYOUT_RENDERING_LOCAL_READ_KHR` │
│ 着色器：`subpassLoad`     │ 着色器：**同样是 `subpassLoad`**（无需改 shader）   │
└──────────────────────────┴───────────────────────────────────────────────────┘
```

**关键优点**：**着色器接口不变**。用 `subpassInput` / `subpassLoad` 写的 shader 可以直接复用，改动全在应用侧。

### 5.2 代码骨架

```cpp
// ═════════════════════════════════════════════════════════════════════════════
// Dynamic Rendering + Local Read（Vulkan 1.4）
// 需要启用：VK_KHR_dynamic_rendering + VK_KHR_dynamic_rendering_local_read
//          feature: dynamicRenderingLocalRead = VK_TRUE
// ═════════════════════════════════════════════════════════════════════════════

// ── 阶段 1：几何（写 G-Buffer）──────────────────────────────────────────────
{
    std::array<vk::RenderingAttachmentInfo, 2> gbufAtt{
        makeAttachment(gbufImage0, vk::ImageLayout::eRenderingLocalRead, /*clear=*/false),
        makeAttachment(gbufImage1, vk::ImageLayout::eRenderingLocalRead, /*clear=*/false),
    };
    // ⚠️ loadOp = DONT_CARE, storeOp = DONT_CARE（G-Buffer 不落主存）

    vk::RenderingInfo ri{};
    ri.setRenderArea(fullArea)
      .setLayerCount(1)
      .setColorAttachments(gbufAtt)
      .setPDepthAttachment(&depthAtt);

    cmd.beginRendering(ri);
    drawOpaqueGeometry(cmd);
    cmd.endRendering();
}

// ── 阶段 2：★ BY_REGION barrier，把 G-Buffer 变成可 local read 的 input ────
{
    vk::ImageMemoryBarrier2 localReadBarrier{};
    localReadBarrier
        .setSrcStageMask(vk::PipelineStageFlagBits2::eColorAttachmentOutput)
        .setSrcAccessMask(vk::AccessFlagBits2::eColorAttachmentWrite)
        .setDstStageMask(vk::PipelineStageFlagBits2::eFragmentShader)
        .setDstAccessMask(vk::AccessFlagBits2::eInputAttachmentRead)
        .setOldLayout(vk::ImageLayout::eRenderingLocalRead)   // ★ 新布局
        .setNewLayout(vk::ImageLayout::eRenderingLocalRead)   // ★ layout 不变
        .setImage(gbufImage0.image())                          // 每个 G-Buffer 都要一次
        .setSubresourceRange(fullRange);

    cmd.pipelineBarrier2(vk::DependencyInfo{}
        .setDependencyFlags(vk::DependencyFlagBits::eByRegion) // ★★★ 必须 BY_REGION
        .setImageMemoryBarriers(localReadBarrier));
    // 对 gbufImage1 重复
}

// ── 阶段 3：光照（读 G-Buffer 作为 input attachment，写最终颜色）────────────
{
    // ★ input attachment 索引映射：告诉驱动哪个 color attachment 变成 input N
    uint32_t colorIndices[1] = { 0 };
    vk::RenderingInputAttachmentIndexInfo localReadInfo{};
    localReadInfo
        .setColorAttachmentCount(1)
        .setPColorAttachmentInputIndices(colorIndices);

    auto finalColor = makeAttachment(swapchainImage,
                                     vk::ImageLayout::eColorAttachmentOptimal,
                                     /*clear=*/true);   // 最终颜色要 Clear

    vk::RenderingInfo ri{};
    ri.setRenderArea(fullArea)
      .setLayerCount(1)
      .setColorAttachments(finalColor)
      .setPNext(&localReadInfo);      // ★ 挂上 local read 信息

    cmd.beginRendering(ri);
    drawFullscreenTriangle(cmd, m_lightingPipeline);  // 全屏 draw 完全无害！
    cmd.endRendering();
}
```

### 5.3 与 subpass 的语义对应

| Subpass 概念 | Dynamic Rendering 对应 |
|-------------|---------------------|
| `vkCmdNextSubpass` | 带 `eByRegion` 的 `vkCmdPipelineBarrier2` |
| `VkAttachmentReference` (input) | `VkRenderingInputAttachmentIndexInfoKHR` |
| `VkAttachmentReference` (color 重映射) | `VkRenderingAttachmentLocationInfoKHR` + `vkCmdSetRenderingAttachmentLocationsKHR` |
| `VK_DEPENDENCY_BY_REGION_BIT` | `vk::DependencyFlagBits::eByRegion`（同样必填） |
| `VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL` (input) | `VK_IMAGE_LAYOUT_RENDERING_LOCAL_READ_KHR` |

### 5.4 「全屏 draw 会不会破坏局部性」的准确回答

⚠️ 这是资料中一个**关键区分点搞错的地方**。准确规则是：

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  判断标准：访问模式，而不是「是否全屏」                                          │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ✅ 【全屏 draw + input attachment / subpassLoad】                            │
│     → 读取被限制在「当前像素位置」                                             │
│     → 硬件可以把数据留在片上                                                  │
│     → 完全无害                                                               │
│                                                                              │
│  ❌ 【全屏 draw + sampler2D 采样 G-Buffer 纹理】                              │
│     → 采样地址任意（虽然这个例子实际就是同位置，但驱动无法证明）                    │
│     → G-Buffer 必须是完整纹理，必须落主存                                      │
│     → 破坏片上闭环                                                            │
│                                                                              │
│  ⚠️ 所以：把光照 Pass 写成全屏 vkCmdDraw 一点问题都没有，                       │
│     前提是它通过 input attachment 读 G-Buffer。                                │
│     真正致命的是改用 sampler2D。                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

**验证方式**：

| 手段 | 说明 |
|------|------|
| `VK_EXT_subpass_merge_feedback` | 直接查询 subpass 是否被合并（路径 A 用） |
| 用 sampler 时观察带宽 | G-Buffer 若落主存，带宽会明显抬升 |
| 检查 `VK_DEPENDENCY_BY_REGION_BIT` | 漏掉它是最常见的失效原因 |
| 检查 G-Buffer 的 `storeOp` | 若是 `STORE` 且之后用 sampler 读，说明本来就没打算片上闭环 |

### 5.5 两条路径的选择建议

| 场景 | 建议路径 |
|------|---------|
| 引擎已深度依赖 `VkRenderPass`（如 FrameGraph 以 render pass 为节点） | 路径 A，但用 `VK_EXT_subpass_merge_feedback` 加断言 |
| 新写的渲染器 / Vulkan 1.3+ 起 | 路径 B（dynamic rendering），未来迁移成本为零 |
| 需要动态切换附件集合（如不同质量档位） | 路径 B（render pass 的组合爆炸是它的固有痛点） |
| 目标平台驱动对 `VK_KHR_dynamic_rendering_local_read` 支持不全 | 路径 A 兜底 |

⚠️ **注意**：路径 B 的 `VK_KHR_dynamic_rendering_local_read` 是较新扩展（2024 年发布），**必须运行时查询支持情况并准备回退路径**。回退逻辑可以是「两个独立 render pass + sampler 采样」，虽然慢但正确。

---

## 6. 移动端 Deferred 的坑点清单

| # | 坑 | 现象 | 对策 |
|---|----|------|------|
| 1 | G-Buffer 用 `RGBA16F` 存法线和位置 | 带宽爆炸、bin 变小、可能触发 Direct Mode | Octahedron 编码 + RG16F；位置由深度重建 |
| 2 | G-Buffer 开 MSAA | 法线被平均、边缘错误混色；片上需求倍增 | 禁用 MSAA，改用 FXAA / SMAA / TAA |
| 3 | subpass 依赖漏掉 `BY_REGION` | 优化完全失效，带宽回到 IMR 水平 | 显式设置；用 `VK_EXT_subpass_merge_feedback` 断言 |
| 4 | G-Buffer 拆成两个独立 render pass | 带宽翻倍 + 额外 layout transition | 合成一个 pass（subpass 或 dynamic rendering + local read） |
| 5 | G-Buffer 设 `storeOp = STORE` | 本应省下的带宽全花出去 | 设 `DONT_CARE`；明确不写回可用 `VK_ATTACHMENT_STORE_OP_NONE_QCOM` |
| 6 | 最终颜色附件用默认 `LOAD_OP_LOAD` | 驱动被迫从主存读回 tile buffer | 设 `CLEAR` 或 `DONT_CARE` |
| 7 | 在光照 shader 里采样当前 pass 的深度附件 | Mali 上触发 late ZS test，**禁用 HSR** | 把线性深度写进 G-Buffer 通道，别采样深度附件 |
| 8 | 光照 shader 里用 `discard` | 强制 late ZS update，降低 early ZS 效率 | 只在必要处用；考虑 alpha-to-coverage 或预乘 alpha |
| 9 | 透明物体走 deferred | 无法正确混合 | 单独 forward pass；透明材质设 `Surface ForwardShading` |
| 10 | 后处理链过长 | Resolve 成本从 1ms 涨到 3ms+ | 合并 pass、降级 Bloom/SSAO、用低分辨率中间 |
| 11 | TAA 的 Motion Vector 挤占 G-Buffer 预算 | 在 Mali 小核上直接顶出片上容量 | 评估是否值得；或降低 MV 精度 / 只在需要时启用 |
| 12 | 帧内中途切 framebuffer 渲染"副产品" | 破坏分箱批处理，Resolve 次数增加 | 把副产品渲染集中到帧的独立阶段，或在帧首完成 |
| 13 | 使用 tessellation / geometry shader | 触发 Adreno Direct Mode | 移动端改用 mesh shader 或 CPU/compute 侧镶嵌 |
| 14 | 顶点着色器里做纹理采样（位移贴图等） | 分箱阶段纹理访问，触发 Direct Mode | 用 compute 预计算到 SSBO，VS 只读 SSBO |
| 15 | 为兼容旧驱动写 `VkRenderPass` 但目标平台不支持 local read | 无法迁移到 dynamic rendering 路径 | 该场景下路径 A 是唯一选择，保留 |

---

## 7. 落地检查表

```
□ 管线选型
    □ 明确记录选择 Forward / Forward+ / Deferred 的**依据**（动态光数量、是否预计算光照、
       是否需要 MSAA、Resolve 预算、目标机型）
    □ 若选 Deferred，透明物体有独立的 forward pass 规划

□ G-Buffer 设计
    □ 每像素总字节数 ≤ 16（含深度）为佳；绝不超过 24
    □ 法线用 Octahedron 编码，不存 WG 全精度法线
    □ 不存 world position
    □ 无 MSAA
    □ 每个字段都能说出"为什么必须存"；说不出的就删

□ 片上闭环
    □ G-Buffer 与光照在同一 render pass（subpass）或同一组带 local read 的 dynamic rendering
    □ subpass 依赖 / barrier 均带 BY_REGION
    □ G-Buffer storeOp = DONT_CARE
    □ 深度 storeOp = DONT_CARE（若后续不用）
    □ 线条颜色附件 loadOp = CLEAR
    □ 用 VK_EXT_subpass_merge_feedback 或带宽实测验证闭环生效

□ 着色器
    □ 光照 shader 用 subpassInput / subpassLoad，不是 sampler2D
    □ 不在位置计算路径上采样纹理
    □ 不写 gl_FragDepth
    □ 不采样当前 pass 的深度附件（Mali）
    □ 减少 discard
    □ 谨慎使用 framebuffer fetch

□ 后处理
    □ 统计整帧被"渲染到"的离屏贴图数量，估算 Resolve 成本
    □ 尽量合并后处理 pass
    □ Bloom 用低分辨率 + 少量 mip
    □ 逐项评估 SSAO / DOF / 运动模糊在目标机型上的性价比
```

---

## 8. 小结

| 要点 | 说明 |
|------|------|
| **「移动端延迟渲染是天作之合」是过强论断** | TBDR 把 deferred 的最大缺点从「致命」降为「可接受」，所以可行；但不等于最优 |
| **两个官方立场方向相反，原因在场景特征** | UE 推荐大多数移动项目用 Deferred；Oculus 明确劝退移动 VR（Resolve 成本从 1ms 涨到 3ms+）。差异来自分辨率、帧预算、光照结构 |
| **决策依据是四件事** | 动态光源数量、材质复杂度、Resolve 预算、是否需要 MSAA 与预计算光照 |
| **G-Buffer 的核心原则是「能重建就不存」** | 位置由深度重建；法线用 Octahedron 编码；roughness 塞进 albedo 的 alpha；目标 ≤ 16 B/px |
| **Deferred 不可 MSAA 的本质是语义冲突** | 非线性着色不能对输入做线性平均。不是单纯的 GMEM 尺寸问题 |
| **`VK_DEPENDENCY_BY_REGION_BIT` 是生命线** | 漏掉它，所有片上优化归零。可用 `VK_EXT_subpass_merge_feedback` 断言 |
| **「全屏 draw」本身无害，采样方式才是关键** | 用 input attachment 的全屏 draw 保持片上闭环；换成 sampler2D 才破坏局部性 |
| **Vulkan 1.4 起 `VkRenderPass` 已 deprecated** | 新项目走 dynamic rendering + `VK_KHR_dynamic_rendering_local_read`；着色器接口不变，改动全在应用侧 |
| **Resolve 是被低估的主线成本** | 每个「渲染到离屏贴图」都有一次；移动端后处理链必须精简 |
| **显式片上内存管理已成为可能** | `VK_QCOM_tile_memory_heap`（2025）允许 G-Buffer 直接分配在 tile memory 并跨 pass 常驻 |

---

*上一篇：[03-逐Tile流水线与GMEM](03-逐Tile流水线与GMEM.md) | 下一篇：[05-性能诊断与优化清单](05-性能诊断与优化清单.md)*
