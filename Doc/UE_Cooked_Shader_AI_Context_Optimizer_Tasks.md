# UE Cooked Shader AI Context Optimizer Tasks

本文档把 `UE_Cooked_Shader_AI_Context_Optimizer_Implementation_Plan.md` 拆成可执行 task。

目标：在已有 `CUE4Parse.ShaderBundleExporter` semantic outputs 基础上，增加变体归约和 AI context pack，让后续 Unity 6 URP Deferred + DOTS instancing 还原 Agent 不需要直接面对全部 cooked shader variants。

默认 golden case：

```text
Game: Subnautica2
Material: /Game/Materials/_Master/Master/M_Character_Teeth
Bundle: D:\ShaderWP\M_Character_Teeth.bundle
Expected shaders: 14
Expected stages: 5 VS / 6 PS / 3 CS
Unity target: Unity 6 URP Deferred + DOTS instancing
```

## Task Status Legend

```text
PENDING      未开始
IN_PROGRESS 进行中
DONE         已完成并通过验收
BLOCKED      被外部条件阻塞
PARTIAL      已有可用输出，但没有达到完整目标
```

## Task 00 - AI Context Optimizer Architecture

Status: `DONE`

Depends on: completed semantic exporter

Goal:

建立 variant reduction / AI context pack 的代码组织、共享模型、manifest/status 接入。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/AIContext/
CUE4Parse/CUE4Parse.ShaderBundleExporter/Models/AIContextModels.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/AIContextAnalysisPipeline.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/SemanticAnalysisPipeline.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/BundleManifestWriter.cs
```

Outputs:

```text
analysis/variant_inventory.json
analysis/shader_similarity_groups.json
analysis/reconstruction_entrypoints.json
analysis/variant_diff_summary.json
analysis/deferred_lighting_policy.json
analysis/lightpass_candidates.json
analysis/lightpass_audit.json
analysis/ai_context_pack.json
analysis/ai_context_pack.md
```

Steps:

1. 新增 `AIContext` exporter folder。
2. 新增 models：

```text
VariantInventoryDocument
ShaderVariantInfo
ShaderVariantScores
ShaderSimilarityGroupsDocument
ReconstructionEntrypointsDocument
VariantDiffSummaryDocument
AiContextPackDocument
```

3. 所有对象支持 `Score` / `Confidence` / `Evidence` / `SourceFiles`。
4. AI context pipeline 可从 full export 和 existing bundle 两种路径运行。
5. `SemanticAnalysisStatus.CompletedPasses` 增加 AI context pass。
6. `manifest.json` 文件列表包含新增 outputs。

Acceptance:

```text
dotnet build succeeds
Default export writes all six AI context output files
manifest.json contains those files
semantic_status.json lists variant/context passes
```

## Task 01 - CLI Flags For Variant And Context Passes

Status: `DONE`

Depends on: Task 00

Goal:

给 exporter 增加显式控制开关，支持 full export、semantic-only 和 context-only。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/ShaderBundleExportOptions.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Program.cs
```

New CLI:

```text
--analyze-variants
--write-ai-context-pack
--no-ai-context-pack
--ai-context-budget <characters>
--max-primary-variants <count>
--max-supporting-variants <count>
--context-only <existing bundle>
--audit-lightpass
```

Behavior:

```text
single export: default on
batch export: default on
semantic-only: rerun variant/context pass after semantic pass
context-only: rerun only variant/context pass from existing bundle JSON
no-semantic-analysis: disables dependent context generation unless required existing files exist
```

Acceptance:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- --help
```

Help includes all new flags.

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- --context-only D:\ShaderWP\M_Character_Teeth.bundle
```

Returns `Verify: OK`.

## Task 02 - Variant Inventory Analyzer

Status: `DONE`

Depends on: Task 00

Goal:

为 bundle 内每个 shader 生成 compact feature summary。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/AIContext/VariantInventoryAnalyzer.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Models/AIContextModels.cs
```

Inputs:

```text
manifest.json
analysis/shader_entries.json
analysis/shader_type_info.json
analysis/dxil_resource_usage.json
analysis/resource_bindings.json
analysis/gbuffer_semantics.json
analysis/runtime_features.json
shaders/*.reflection.json
```

Outputs:

```text
analysis/variant_inventory.json
```

Steps:

1. 读取 shader entries。
2. 读取 DXIL usage。
3. 汇总每个 shader 的：

```text
Stage
ShaderModel
RoleCandidate
InstructionCount
TextureOperations count
TextureRegisters
SamplerRegisters
CBufferLoads count
CBufferRegisters
UavOperations count
UavRegisters
MrtOutputs
RuntimeFeatureHints
```

4. 写 `VariantKind` 初始值：

```text
surface_candidate
supporting_vertex
runtime_only
unknown
```

5. 所有 source path 使用 bundle-relative path。

Acceptance:

Golden case：

```text
analysis/variant_inventory.json exists
Shaders count == 14
6 Pixel Shader entries
5 Vertex Shader entries
3 Compute Shader entries
each entry has FeatureCounts
each entry has ResourceSignature
each entry has Evidence array
```

## Task 03 - Shader Reconstruction Scorer

Status: `DONE`

Depends on: Task 02

Goal:

给每个 shader 计算 surface reconstruction relevance score，支持 deterministic selection。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/AIContext/ShaderReconstructionScorer.cs
```

Outputs:

```text
analysis/variant_inventory.json
```

Scoring fields:

```text
SurfaceRelevance
Completeness
RuntimePenalty
RedundancyPenalty
FinalSelectionScore
```

Steps:

1. Pixel shader 加权：stage、MRT、texture ops、semantic binding overlap、cbuffer loads。
2. Vertex shader 加权：stage、signature usefulness、group proximity。
3. Compute shader 默认降低 reconstruction score。
4. UAV / VT feedback / page-table / renderer-only hints 增加 penalty。
5. 每个 score 写 reason。

Acceptance:

Golden case：

```text
All shaders have Scores
All score values are 0..1
At least one Pixel Shader has FinalSelectionScore > 0.5
Compute Shader scores are lower than selected primary pixel shaders
Runtime-heavy shaders include RuntimePenalty evidence
```

## Task 04 - Shader Similarity Grouping

Status: `DONE`

Depends on: Task 02, Task 03

Goal:

把相似 permutation 分组成 representative + members，减少 AI 需要读取的 shader 数。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/AIContext/ShaderSimilarityGrouper.cs
```

Inputs:

```text
analysis/variant_inventory.json
```

Outputs:

```text
analysis/shader_similarity_groups.json
```

Steps:

1. 构造 exact signature：

```text
stage|role|textureRegisters|samplerRegisters|cbufferRegisters|uavRegisters|targets|runtimeFlags|shaderModel
```

2. 对 exact signature 相同的 shader 分组。
3. 对同 stage / same runtime classification 的 shader 做 fuzzy grouping：

```text
Jaccard(TextureRegisters) >= 0.80
Jaccard(CBufferRegisters) >= 0.80
TextureOps difference <= 20%
CBufferLoads difference <= 20%
```

4. 每组选择 representative：

```text
highest FinalSelectionScore
then highest TextureOperations
then highest MrtOutputs
then highest CBufferLoads
then lowest RuntimePenalty
```

5. 写 SimilarityBasis 和 RepresentativeReason。

Acceptance:

```text
analysis/shader_similarity_groups.json exists
Every shader appears in exactly one group
Every group has RepresentativeShaderFile
RepresentativeShaderFile is one of Members
Every group has Confidence and Evidence
```

## Task 05 - Reconstruction Entrypoint Selector

Status: `DONE`

Depends on: Task 03, Task 04

Goal:

生成给 Unity Agent 的主入口选择：读哪些 shader，哪些不读，哪些抽象。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/AIContext/ReconstructionEntrypointSelector.cs
```

Inputs:

```text
analysis/variant_inventory.json
analysis/shader_similarity_groups.json
analysis/runtime_features.json
analysis/semantic_binding_map.json
```

Outputs:

```text
analysis/reconstruction_entrypoints.json
```

Steps:

1. 选出 primary pixel shader representatives。
2. 默认最多 2 个 primary pixel shaders。
3. 选出 supporting vertex shaders。
4. 默认最多 2 个 supporting vertex shaders。
5. Compute / UAV-heavy / VT feedback / page-table shader 放入 `RuntimeOnlyOrAbstract`。
6. 相似组非代表成员放入 `IgnoredSimilarVariants`。
7. 如果无法置信选择 primary PS，写 `KnownRisks` 并让 verifier 失败。

Acceptance:

Golden case：

```text
analysis/reconstruction_entrypoints.json exists
PrimaryPixelShaders count >= 1
PrimaryPixelShaders count <= 2
SupportingVertexShaders count <= 2
RuntimeOnlyOrAbstract count >= 1
IgnoredSimilarVariants exists
Every selected ShaderFile exists
Every entry has Reason and Evidence
```

## Task 06 - Variant Diff Summary

Status: `DONE`

Depends on: Task 04, Task 05

Goal:

给 AI Agent 解释哪些变体被合并、差异是什么、为什么可以先不读。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/AIContext/VariantDiffSummarizer.cs
```

Outputs:

```text
analysis/variant_diff_summary.json
```

Steps:

1. 汇总 total shader count / group count / selected count。
2. 对每组输出 common features。
3. 对每组输出 differences。
4. 给每组 recommendation：

```text
read_representative_only
read_all
abstract
```

Acceptance:

```text
analysis/variant_diff_summary.json exists
Summary.TotalShaders == 14 for golden case
Every similarity group has a diff summary
At least one group recommendation is read_representative_only or abstract
```

## Task 07 - Deferred Lighting Reuse Policy

Status: `DONE`

Depends on: Task 05, completed semantic exporter

Goal:

明确 Deferred 还原边界：Unity shader 还原只覆盖 Material/BasePass -> surface/GBuffer 逻辑，Light Pass 默认复用 Unity URP Deferred，不把 UE Deferred LightPass 搬进普通 material shader。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/AIContext/DeferredLightingPolicyWriter.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Models/AIContextModels.cs
```

Outputs:

```text
analysis/deferred_lighting_policy.json
```

Steps:

1. 写固定策略：

```text
LightingPassStrategy = reuse_unity_urp_deferred_lighting
```

2. 明确 `ReconstructScope`：

```text
texture sampling
material parameter application
normal reconstruction
surface attribute calculation
GBuffer/deferred output mapping
```

3. 明确 `DoNotReconstruct`：

```text
UE deferred light accumulation
UE shadow/light grid code
UE reflection environment lighting
UE renderer-only VT feedback/page-table behavior
```

4. 如果 `material_parameters.json` 中存在 MSM_Subsurface / Subsurface hints，写风险：

```text
URP Deferred does not provide byte-equivalent UE MSM_Subsurface.
Use documented approximation or custom pass requirement.
```

5. 输出 Unity strategy：

```text
stock-like light pass -> reuse Unity URP Deferred lighting
custom light pass -> do not port to material shader; emit RendererFeature/fullscreen pass/custom BRDF requirement
```

Acceptance:

```text
analysis/deferred_lighting_policy.json exists
LightingPassStrategy == reuse_unity_urp_deferred_lighting
ReconstructScope includes gbuffer_or_deferred_output_mapping
DoNotReconstruct includes ue_deferred_light_accumulation
UnityStrategy exists
Risks exists
Evidence array exists
```

## Task 08 - LightPass Candidate Scan And Audit

Status: `DONE`

Depends on: Task 07, completed semantic exporter

Goal:

轻量审计原游戏 shader archive / `.stinfo` / metadata 中是否存在 stock UE LightPass 或 project/plugin custom LightPass 信号。审计结果只影响 Unity lighting strategy，不扩大 material shader 还原范围。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/AIContext/LightPassCandidateScanner.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/AIContext/LightPassAuditor.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Models/AIContextModels.cs
```

Inputs:

```text
source/shader_type_info/*.stinfo
analysis/shader_type_info.json
source/shader_archive.metadata.json
analysis/runtime_features.json
analysis/gbuffer_semantics.json
analysis/dxil_resource_usage.json
```

Outputs:

```text
analysis/lightpass_candidates.json
analysis/lightpass_audit.json
```

Scan terms:

```text
DeferredLight
DeferredLighting
LightGrid
Clustered
ReflectionEnvironment
Subsurface
ScreenSpace
Lumen
Shadow
RectLight
SkyLight
BRDF
GBuffer
```

Origin classification:

```text
stock_engine: Engine/Shaders/Private/* or Engine/Shaders/Public/*
project_custom: Project/Shaders/*, /Game/Shaders/*, Source/*/Shaders/*
plugin_custom: Plugins/*/Shaders/*
unknown: no reliable path evidence
```

Audit conclusion:

```text
stock_like
custom_lighting_detected
inconclusive
```

Recommended Unity strategy:

```text
reuse_unity_urp_deferred_lighting
reuse_plus_custom_renderer_feature
requires_manual_renderer_review
```

Steps:

1. 从 `.stinfo` readable strings / shader type info candidate strings 中扫描 lightpass 关键词。
2. 从 shader archive metadata 中扫描 source path / shader name candidates。
3. 根据 source path 判断 stock engine / project custom / plugin custom。
4. 结合 runtime features 和 GBuffer semantics 判断 renderer-only shader signals。
5. 输出 `DoNotPortIntoMaterialShader` 列表。
6. 如果发现 project/plugin custom lighting，输出 `CustomLightingRequirement.Required = true`。
7. 如果无法证明 stock 或 custom，输出 `Conclusion = inconclusive`，并建议 manual renderer review。

Acceptance:

```text
analysis/lightpass_candidates.json exists
analysis/lightpass_audit.json exists
lightpass_audit Conclusion is stock_like/custom_lighting_detected/inconclusive
lightpass_audit RecommendedUnityStrategy exists
lightpass_audit DoNotPortIntoMaterialShader exists
lightpass_audit never recommends porting UE LightPass into Unity material shader
custom project/plugin signals, if found, are listed in CustomSignals
```

## Task 09 - AI Context Pack JSON

Status: `DONE`

Depends on: Task 05, Task 06, Task 07, Task 08

Goal:

生成给后续 Unity Agent 的首要 JSON 输入，避免 Agent 直接读完整 DXIL。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/AIContext/AiContextPackWriter.cs
```

Inputs:

```text
analysis/unity_deferred_reconstruction_contract.json
analysis/semantic_binding_map.json
analysis/reconstruction_entrypoints.json
analysis/runtime_features.json
analysis/variant_diff_summary.json
analysis/deferred_lighting_policy.json
analysis/lightpass_audit.json
analysis/material_function_dependencies.json
analysis/texture_assets.json
parameters/material_parameters.json
parameters/textures.json
```

Outputs:

```text
analysis/ai_context_pack.json
```

Steps:

1. 写 Target：

```text
Unity 6
URP
Deferred
DOTS instancing true
```

2. 写 PrimaryInputs。
3. 写 RecommendedReadOrder。
4. 写 SelectedShaders。
5. 写 TextureBindings / UniformBindings。
6. 写 RuntimeFeaturesToAbstract。
7. 写 DeferredLightingPolicy。
8. 写 LightPassAudit。
9. 写 GBufferCandidates。
10. 写 MaterialFunctionContext。
11. 写 KnownUnknowns。
12. 写 DoNotReadByDefault。
13. 写 EvidenceDrillDown。
14. 不 inline 完整 `.dxil.ll`。

Acceptance:

```text
analysis/ai_context_pack.json exists
Target.RenderingPath == Deferred
Target.DotsInstancing == true
SelectedShaders includes PrimaryPixelShaders
DeferredLightingPolicy.LightingPassStrategy == reuse_unity_urp_deferred_lighting
LightPassAudit.RecommendedUnityStrategy exists
DoNotReadByDefault includes raw .dxil.ll or ignored variants
JSON file size stays bounded by --ai-context-budget unless explicitly disabled
```

## Task 10 - AI Context Pack Markdown

Status: `DONE`

Depends on: Task 09

Goal:

生成可直接给 AI Agent 阅读的 Markdown 摘要。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/AIContext/AiContextPackMarkdownWriter.cs
```

Outputs:

```text
analysis/ai_context_pack.md
```

Required sections:

```text
# AI Context Pack
## Verify First
## Primary Read Order
## Selected Shader Entrypoints
## Texture And Channel Map
## Uniform Candidates
## Runtime Features To Abstract
## Deferred Lighting Policy
## LightPass Audit
## GBuffer / Deferred Notes
## Material Function Context
## Evidence Drill Down
## Known Unknowns
## Do Not Read By Default
```

Acceptance:

```text
analysis/ai_context_pack.md exists
mentions verify_bundle.ps1
mentions unity_deferred_reconstruction_contract.json
mentions semantic_binding_map.json
mentions reconstruction_entrypoints.json
mentions deferred_lighting_policy.json
mentions lightpass_audit.json
mentions reusing Unity URP Deferred lighting instead of porting UE LightPass into a material shader
mentions not reading all *.dxil.ll by default
```

## Task 11 - Context-Only Offline Mode

Status: `DONE`

Depends on: Task 01, Task 02-10

Goal:

支持对已有 bundle 只重新生成 variant/context outputs，不重新解包 shader。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Program.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/AIContextAnalysisPipeline.cs
```

Command:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- --context-only D:\ShaderWP\M_Character_Teeth.bundle
```

Steps:

1. 读取 existing `manifest.json`。
2. 读取 existing semantic JSON。
3. 不访问 paks / mapping / dxc / decompress_shader。
4. 重写 6 个 AI context outputs。
5. 更新 manifest file list 和 semantic_status。
6. 自动运行 verify。

Acceptance:

```text
command returns exit code 0
command prints Verify: OK
modified files limited to analysis/*.json, analysis/*.md, manifest.json, semantic_status.json
```

## Task 12 - Verify AI Context Outputs

Status: `DONE`

Depends on: Task 02-11

Goal:

扩展 `--verify-only`，校验 AI context outputs 的结构和引用完整性。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/BundleVerifier.cs
```

Checks:

```text
variant_inventory parseable
shader_similarity_groups parseable
reconstruction_entrypoints parseable
variant_diff_summary parseable
deferred_lighting_policy parseable
lightpass_candidates parseable
lightpass_audit parseable
ai_context_pack.json parseable
ai_context_pack.md exists
all selected ShaderFile paths exist
all ignored variants point to existing representative
all scores/confidence values are 0..1
primary pixel shader count <= max
selected primary shaders are Pixel Shader
deferred_lighting_policy LightingPassStrategy == reuse_unity_urp_deferred_lighting
lightpass_audit does not recommend porting UE LightPass into Unity material shader
ai_context_pack does not inline full .dxil.ll text
```

Acceptance:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- --verify-only D:\ShaderWP\M_Character_Teeth.bundle
```

Returns `Verify: OK`.

## Task 13 - Golden Case Regeneration

Status: `DONE`

Depends on: Task 12

Goal:

用完整 exporter 重新生成 `D:\ShaderWP\M_Character_Teeth.bundle`，验证变体归约输出。

Command:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --game Subnautica2 `
  --paks "D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\Subnautica2\Content\Paks" `
  --mapping "D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\5.6.1-114707+++Project+SN2-Release-Hotfix-Live-Subnautica2.usmap" `
  --material "/Game/Materials/_Master/Master/M_Character_Teeth" `
  --out "D:\ShaderWP\M_Character_Teeth.bundle" `
  --decompress-shader "D:\Github\UEShaderMapExtractor\Build\decompress_shader.exe" `
  --dxc "C:\Program Files (x86)\Windows Kits\10\bin\10.0.22621.0\x64\dxc.exe" `
  --overwrite `
  --semantic-analysis `
  --analyze-variants `
  --write-ai-context-pack `
  --audit-lightpass
```

Acceptance:

```text
Base semantic outputs still exist
analysis/variant_inventory.json exists
analysis/shader_similarity_groups.json exists
analysis/reconstruction_entrypoints.json exists
analysis/variant_diff_summary.json exists
analysis/deferred_lighting_policy.json exists
analysis/lightpass_candidates.json exists
analysis/lightpass_audit.json exists
analysis/ai_context_pack.json exists
analysis/ai_context_pack.md exists
variant_inventory Shaders count == 14
PrimaryPixelShaders count >= 1
PrimaryPixelShaders count <= 2
RuntimeOnlyOrAbstract count >= 1
deferred_lighting_policy LightingPassStrategy == reuse_unity_urp_deferred_lighting
lightpass_audit RecommendedUnityStrategy exists
verify_bundle.ps1 returns Verify: OK
```

## Task 14 - Update ShaderWP Agent Docs

Status: `DONE`

Depends on: Task 13

Goal:

更新 `D:\ShaderWP` 的 AI Agent 文档，让新会话优先读 AI context pack，而不是直接读全部 semantic JSON 或 DXIL。

Files:

```text
D:\ShaderWP\README.md
D:\ShaderWP\AGENTS.md
D:\ShaderWP\WORKFLOW.md
D:\ShaderWP\NEXT_TASK.md
D:\ShaderWP\PROMPT_NEXT_SESSION.md
D:\ShaderWP\agent_context.json
D:\ShaderWP\skills\unity6-urp-shader-reconstruction\SKILL.md
```

New read order:

```text
1. verify_bundle.ps1
2. analysis/ai_context_pack.md
3. analysis/ai_context_pack.json
4. analysis/reconstruction_entrypoints.json
5. analysis/deferred_lighting_policy.json
6. analysis/lightpass_audit.json
7. analysis/unity_deferred_reconstruction_contract.json
8. analysis/semantic_binding_map.json
9. DXIL drill-down only when needed
```

Acceptance:

新会话只读 `PROMPT_NEXT_SESSION.md`，即可知道：

```text
先 verify bundle
先读 ai_context_pack.md/json
再读 reconstruction_entrypoints.json
再读 deferred_lighting_policy.json / lightpass_audit.json
再读 contract / semantic_binding_map
不要默认读取全部 *.dxil.ll
输出 Unity6URP files
目标 Deferred + DOTS instancing
```

## Task 15 - Per-export Agent workspace docs

Status: DONE

目的：每次 shader bundle 导出都必须自带 AI Agent 可读的入口文档、工作流、skill 和命令脚本，不能只在某个手工整理的 `D:\ShaderWP` 根目录保留这些文件。

Implemented files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/AgentWorkspaceWriter.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/ShaderBundleExporter.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Program.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/BundleVerifier.cs
AGENTS.md
```

Required single-bundle outputs:

```text
README.md
AGENTS.md
WORKFLOW.md
NEXT_TASK.md
PROMPT_NEXT_SESSION.md
agent_context.json
skills/unity6-urp-deferred-shader-reconstruction/SKILL.md
commands/verify_bundle.ps1
commands/refresh_context.ps1
commands/export_again.ps1
```

Behavior:

```text
full export writes Agent workspace docs after manifest generation
--context-only repairs Agent workspace docs for existing bundles
--semantic-only repairs Agent workspace docs for existing bundles
batch export writes root-level batch Agent docs and per-bundle docs
manifest.json includes AgentWorkspace section
manifest Files list includes generated docs/scripts/skill
verify-only requires Agent workspace files
```

Acceptance:

```text
D:\ShaderWP\NewTest\README.md exists
D:\ShaderWP\NewTest\AGENTS.md exists
D:\ShaderWP\NewTest\WORKFLOW.md exists
D:\ShaderWP\NewTest\NEXT_TASK.md exists
D:\ShaderWP\NewTest\PROMPT_NEXT_SESSION.md exists
D:\ShaderWP\NewTest\agent_context.json exists
D:\ShaderWP\NewTest\skills\unity6-urp-deferred-shader-reconstruction\SKILL.md exists
D:\ShaderWP\NewTest\commands\verify_bundle.ps1 exists
D:\ShaderWP\NewTest\commands\refresh_context.ps1 exists
D:\ShaderWP\NewTest\commands\export_again.ps1 exists
agent_context.json Target.RenderingPath == Deferred
agent_context.json Target.DotsInstancing == true
verify-only returns Verify: OK
```

Verified commands:

```powershell
dotnet build CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release

dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- --context-only D:\ShaderWP\NewTest --verbose
```

Observed result:

```text
dotnet build: Build succeeded, 0 Warning(s), 0 Error(s)
context-only: Agent workspace docs: updated
context-only: Verify: OK
```

## Completion Evidence

Status as of 2026-06-04: Task 00-14 已在 `CUE4Parse/CUE4Parse.ShaderBundleExporter` 实现并通过 golden case 验证。

Verified commands:

```powershell
dotnet build CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release

dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- --context-only D:\ShaderWP\M_Character_Teeth.bundle --verbose

dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- --verify-only D:\ShaderWP\M_Character_Teeth.bundle --verbose

D:\ShaderWP\commands\verify_bundle.ps1
```

Observed results:

```text
dotnet build: Build succeeded
context-only: Verify: OK
verify-only: Verify: OK
verify_bundle.ps1: Verify: OK
analysis/semantic_status.json Status: success
AI context completed passes: variant-inventory, reconstruction-entrypoints, variant-diff-summary, deferred-lighting-policy, lightpass-candidates, lightpass-audit, ai-context-pack, ai-context-pack-markdown
variant_inventory Shaders: 14
Stage summary: 6 Pixel Shader / 3 Compute Shader / 5 Vertex Shader
PrimaryPixelShaders: 2
RuntimeOnlyOrAbstract: 3
IgnoredSimilarVariants: 6
analysis/deferred_lighting_policy.json LightingPassStrategy: reuse_unity_urp_deferred_lighting
analysis/lightpass_audit.json Conclusion: inconclusive
analysis/lightpass_audit.json RecommendedUnityStrategy: requires_manual_renderer_review
analysis/ai_context_pack.json Target: Unity 6 URP Deferred, DOTS instancing true
```

## Suggested Execution Order

```text
Task 00
Task 01
Task 02
Task 03
Task 04
Task 05
Task 06
Task 07
Task 08
Task 09
Task 10
Task 11
Task 12
Task 13
Task 14
Task 15
```

## Minimal Useful Milestone

如果不能一次完成全部任务，最小有用里程碑是：

```text
Task 02 - Variant inventory
Task 03 - Shader reconstruction scorer
Task 05 - Reconstruction entrypoint selector
Task 07 - Deferred lighting reuse policy
Task 08 - LightPass candidate scan and audit
Task 09 - AI context pack JSON
Task 10 - AI context pack Markdown
Task 12 - Verify AI context outputs
```

完成后 Unity Agent 至少能得到：

```text
which shaders to read first
which variants to ignore by default
which runtime-only shaders to abstract
whether UE LightPass appears stock-like/custom/inconclusive
why Unity should reuse URP Deferred lighting by default
compact material/texture/uniform context
DXIL evidence drill-down file references
```
