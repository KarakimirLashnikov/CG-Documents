# 阶段二（M4–M6）— Shader体系与光照材质

> 本文档是「资深引擎开发成长路线」系列文档的第 **2** 篇。覆盖能力域 **B（Shader 与材质体系）**：Shader 编译链路与变体治理、材质数据流布局与反射机制、工业级光照材质方案评估。资深标志：能维护数千个 Shader 变体而不失控，能为项目选型并落地光照方案。
>
> ⚠️ 版本基准 UE 5.8.2；5.8 官方宣称着色器编译次数较此前降低约 68%（State of Unreal 2026），相关 cvar 默认值以本地源码为准。

---

## 1. 阶段目标与验收

| 目标 | 验收标准 |
|------|---------|
| 材质→HLSL→PSO 全链路 | 能讲清材质节点图如何生成 HLSL、如何进 ShaderMap、如何烘焙成 PSO |
| Shader 变体治理 | 建成变体统计/裁剪/预热工具链，给出量化收益 |
| 数据流布局与反射 | 掌握 Shader 参数的绑定机制（`FShaderParameterMap`/`FRHIUniformBufferLayout`），能对接反射工具 |
| 工业级光照材质方案 | 完成 Lumen / 烘焙(Lightmass/GPU Lightmass) / MegaLights 三方案双维度对比报告 |

---

## 2. M4：编译链路与自定义 Shader 类型

### 2.1 全链路图（必须能默画）

```
材质编辑器节点图
      │  FMaterial::Compile → FHLSLMaterialTranslator
      ▼
生成 HLSL（MaterialTemplate.ush 填洞）
      │  按 (VertexFactory × Material × ShaderType × 平台宏) 组合
      ▼
Shader 排列（Permutation）──► FShaderCompilerWorker（多进程编译）
      │  D3D: DXC/FXC    Vulkan: HLSL → DXC → SPIR-V
      ▼
FShaderMap（按 FShaderType/FVertexFactoryType 索引）
      │  Cook 阶段：序列化进包体；运行时：PSO 创建（可异步）
      ▼
PSO 缓存（.upipelinecache / CollectedPSOs）──► 命中 → 零卡顿
```

### 2.2 周计划

| 周 | 主题 | 动作 | 产出 |
|----|------|------|------|
| W13 | 材质代码生成 | 跟读 `FHLSLMaterialTranslator`；用 `r.DumpMaterialDebug` 类手段导出生成 HLSL | 生成代码标注笔记 |
| W14 | Shader 类型体系 | `FGlobalShader` vs `FMaterialShader` vs `FMeshMaterialShader` 职责与注册宏 | 类型体系表（复用 `UE5Shader系统文档/02`） |
| W15 | 编译管线 | `FShaderCompilingManager`、worker 进程、错误处理链路 | 编译流水线笔记 |
| W16 | **落地**：自定义 Shader 类型 | 实现一个自定义 GlobalShader + MeshMaterialShader（教材：`UE5Shader系统文档/04、05`） | 里程碑 M4：可运行 + 链路文档 |

### 2.3 ⚠️ 高频坑

| 坑 | 对策 |
|----|------|
| 改 `.usf/.ush` 不生效 | 共享 Shader 目录缓存问题；改公共头会触发**全量重编译**（数千个），用 `r.ShaderCompiler.EmitWarningsOnLoad` 等手段缩小验证面 |
| `IMPLEMENT_SHADER_TYPE` 模板参数错位 | 编译报符号找不到；逐项核对模板 4 参数与 `ShaderSource` 路径 |
| VertexFactory 排列爆炸 | 一个 VF 改动 = 全 VF × 全材质重编；实验期用极简材质集验证 |
| 编辑器 PSO 卡顿误判 | 编辑器不代表 runtime 卡顿；以打包后运行数据为准 |

---

## 3. M5：变体治理与参数数据流（工程化重点）

### 3.1 Shader 变体治理工具链（里程碑 M5）

```
统计                          裁剪                        预热
────                          ────                        ────
r.ShaderCompiler.        ShaderCookStats /           PSO Collection：
LogShaderCompileTimes    -csv 导出                     采集 gameplay 实际
      │                  按 VF/平台/quality           用到的 PSO
      ▼                  维度透视                        │
变体数量报表              剔除规则：                      ▼
(按类型TopN)             ShouldCache 重写/            打包进
      │                  FShaderType::                 shaderlibrary
      ▼                  ModifyCompilationEnv          │
基线数据 ──────────►  编译时间/包体对比 ◄──────────────┘
                     （裁剪前后 + 预热前后）
```

关键代码骨架（裁剪入口）：

```cpp
// 在自定义 FShaderType 派生类中裁剪排列
static bool ShouldCompilePermutation(
    const FGlobalShaderPermutationParameters& Parameters)
{
    // ⚠️ 裁剪条件必须「平台确定」——引用运行时才决定的状态会导致 PSO miss
    if (IsFeatureLevelSupported(Parameters.Platform, ERHIFeatureLevel::SM5)
        == false)
    {
        return false; // 老平台不编
    }
    // 示例：移动端裁剪掉 RayTracing 相关排列
    return true;
}
```

### 3.2 参数数据流与反射（能力域 B 核心）

理解三级参数绑定机制，并能与自研工具链（`Shader工具链文档/06-实战-反射驱动的布局与描述符生成`）对照：

| 层级 | UE 机制 | 对照（Vulkan/自研） |
|------|---------|---------------------|
| Shader 侧声明 | `BEGIN_SHADER_PARAMETER_STRUCT` + `SHADER_PARAMETER_*` 宏 | GLSL/SPIR-V layout 声明 |
| CPU 侧绑定 | `FShaderParameterMap` / `FRHIUniformBufferLayout`（编译期生成偏移表） | SPIRV-Reflect 反射出 offset |
| 频度分层 | `Scene` / `View` / `Primitive` / `Material` UniformBuffer | set 0~3 按频度分层 |

**实验（W19–W20）**：
- 写一个离线小工具：解析 UE Cook 产物中的 Shader 元数据 + 用 SPIRV-Reflect 解析 Vulkan 版 SPIR-V，**双向核对** UniformBuffer 成员偏移是否一致；
- 产出：数据流布局文档 + 工具仓库；里程碑 M5 附裁剪前后编译时间/包体数字对比。

### 3.3 周计划

| 周 | 动作 | 产出 |
|----|------|------|
| W17 | 变体统计基线（数量/编译时间/包体） | 基线报表 |
| W18 | 实现裁剪规则 + 回归验证画面正确性 | 裁剪 PR + 对比数据 |
| W19 | 参数绑定机制精读 + 双向核对工具 | 工具 v0.1 |
| W20 | PSO 采集与预热流程跑通；**里程碑 M5** | 治理工具链 + 量化报告 |

---

## 4. M6：工业级光照材质方案（评估与落地）

### 4.1 三大光照方案对比框架

不止「谁效果好」，而是按**生产约束**建评估矩阵（里程碑 M6 报告骨架）：

| 维度 | Lumen（HW/SW RT） | 烘焙（GPU Lightmass） | MegaLights（5.8 生产就绪） |
|------|------------------|----------------------|---------------------------|
| 动态场景适应 | ✅ 全自动 | ❌ 静态假设 | ✅ 数千动态投影灯固定开销 |
| 性能开销特征 | 高 GPU、随分辨率/反弹数 | 运行时近零、构建期高 | 固定成本 + 灯数弱相关 |
| 美术工作流 | 即时迭代 | 烘焙等待、UV/lightmap 约束 | 即时迭代 |
| 硬件门槛 | SW RT 可降档；HW RT 需支持 | 无 | 以官方文档为准（5.8 新特性） |
| 移动端 | Lumen Lite（5.8 新增，Switch2/移动） | 主流选择 | ❌ |
| 典型风险 | 噪点/漏光/薄几何 | 动态物体脱节 | 阴影质量与灯密度权衡 |

### 4.2 对比实验设计（同一测试场景，三个方案各跑一遍）

```
场景：室内+室外混合关卡（含动态物体 50 个、动态灯 20 盏）
指标：GPU 帧时间（分 Pass 拆解）/ 显存 / 画质（截图对比组）
     / 美术迭代耗时（改灯→看到结果的秒数）
平台：PC (RTX 3050 可用，但标注低端卡结论外推风险)
工具：Unreal Insights + RenderDoc + stat gpu/unit
```

### 4.3 材质工业实践补充（W24）

- Substrate（5.x 分层材质框架）概念精读：BSDF 分层 vs 传统 ShadingModel 插槽——教材 `UE渲染源码导读文档/03-核心概念Substrate与MegaLights`；
- 材质性能审计清单：指令数 / 采样数 / 动态分支 / VirtualTexture 采样放大；`stat shaderrendering`、材质编辑器内置统计；
- **里程碑 M6 产出**：光照方案对比报告（含选型决策树：给项目类型 → 推荐方案 → 兜底方案）。

---

## 5. 本阶段资料清单

| 资料 | 用途 |
|------|------|
| `UE5Shader系统文档/` 全系列 | M4 主教材 |
| `Shader工具链文档/` 01/02/06 篇 | M5 反射对照 |
| Epic 官方文档：Material Compiling / PSO Caching / Substrate | 语义核对 |
| State of Unreal 2026 / UE 5.8 Release Notes | MegaLights、Lumen Lite、编译优化现状 |
| GPU Lightmass 官方文档 | 烘焙方案细节 |

---

## 6. 小结

| 要点 | 说明 |
|------|------|
| M4 | 材质→HLSL→ShaderMap→PSO 全链路默画级掌握 + 自定义 Shader 类型落地 |
| M5 | 变体治理三段式：统计基线 → 裁剪 → PSO 预热，全部量化 |
| M5 | 参数三级绑定机制与 SPIRV-Reflect 双向核对 |
| M6 | Lumen / 烘焙 / MegaLights 同场景对比实验 + 选型决策树 |
| 最大坑 | 公共 ush 改动触发全量重编；裁剪条件引用运行时状态导致 PSO miss |
| 验收 | 工具链可复用，报告有数字，方案能选型 |

*上一篇：[01-阶段一-UE渲染管线与RHI定制](01-阶段一-UE渲染管线与RHI定制.md) | 下一篇：[03-阶段三-性能分析与优化方法论](03-阶段三-性能分析与优化方法论.md)*
