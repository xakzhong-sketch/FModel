# UE Cooked Shader AI Context Optimizer Implementation Plan

本文档定义 `CUE4Parse.ShaderBundleExporter` 的下一阶段优化：在已有 semantic exporter 基础上增加“变体归约 + AI context pack”能力。

目标是解决 UE cooked shader 与 Unity unpacked shader 类似的变体膨胀问题：一个 material 对应多个 shader / permutation / runtime pass，如果直接把所有 `.dxil.ll` 交给 AI Agent，会造成上下文过大、主逻辑被噪声稀释、还原稳定性下降。

## Current Baseline

当前 golden case：

```text
Game: Subnautica2
Material: /Game/Materials/_Master/Master/M_Character_Teeth
Bundle: D:\ShaderWP\M_Character_Teeth.bundle
Shaders: 14
Stages: 5 Vertex Shader / 6 Pixel Shader / 3 Compute Shader
Semantic status: success
Unity target: Unity 6 URP Deferred + DOTS instancing
```

当前 exporter 已输出：

```text
analysis/unity_deferred_reconstruction_contract.json
analysis/semantic_binding_map.json
analysis/runtime_features.json
analysis/gbuffer_semantics.json
analysis/dxil_resource_usage.json
analysis/uniform_buffer_usage.json
analysis/texture_register_candidates.json
analysis/texture_channel_semantics.json
analysis/shader_type_info.json
shaders/*.dxil.ll
shaders/*.reflection.json
```

这些文件已经把“事实”和“候选推断”分开，但还没有自动回答：

```text
哪些 shader 是 Unity 表面/Deferred 还原时最该看的主变体？
哪些 shader 只是相似 permutation，可以忽略或合并？
哪些 shader 是 VT/UAV/compute/runtime-only，应抽象而不是移植？
给 AI Agent 的最小上下文包应该包含哪些内容？
```

## Problem Statement

UE cooked material 不是单 shader，而是一组 shader/permutation。变体来源包括：

```text
stage: vertex / pixel / compute
pass role: BasePass / depth / velocity / runtime feature
vertex factory
static switch / quality / platform permutation
subsurface / virtual texture / feedback UAV
UE renderer runtime support code
```

对 AI Agent 来说，直接读取全部 `*.dxil.ll` 的问题是：

1. 上下文预算被低价值变体消耗。
2. 多个相似 pixel shader 的细微差异会干扰主逻辑判断。
3. compute / VT / feedback UAV 逻辑容易被误当成 material surface logic。
4. AI 需要重复做筛选，且每次会话结果不稳定。

## Goals

新增 exporter pass，自动生成 AI-friendly variant reduction outputs：

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

核心目标：

1. 为每个 shader 生成结构化 feature summary。
2. 对相似 shader/permutation 分组。
3. 自动选择 1-2 个最适合还原的 primary pixel shader。
4. 自动选择少量 supporting vertex shader。
5. 明确列出 runtime-only / ignore-or-abstract shader。
6. 输出给 Unity Agent 的精简上下文包，默认不包含完整 `.ll`。
7. 所有选择必须有 score、reason、evidence、source files。

## Non-Goals

不做这些事：

```text
不声称自动恢复 UE 源级 material graph
不删除原始 DXIL 或 .ll 文件
不把 compute / VT / UAV runtime code 硬移植到 Unity material shader
不替代 semantic_binding_map 或 unity_deferred_reconstruction_contract
不根据文件名写死 M_Character_Teeth 特例
```

## Target Output Layout

新增文件：

```text
analysis/variant_inventory.json
analysis/shader_similarity_groups.json
analysis/reconstruction_entrypoints.json
analysis/variant_diff_summary.json
analysis/ai_context_pack.json
analysis/ai_context_pack.md
```

`manifest.json` 的 `SemanticAnalysis.Files` 应包含这些文件。

`analysis/semantic_status.json` 应增加 completed pass：

```text
variant-inventory
shader-similarity-groups
reconstruction-entrypoints
variant-diff-summary
deferred-lighting-policy
lightpass-candidates
lightpass-audit
ai-context-pack
```

## Phase 1 - Variant Inventory

新增 `VariantInventoryAnalyzer`。

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

Output:

```text
analysis/variant_inventory.json
```

Schema:

```json
{
  "Material": "/Game/Materials/_Master/Master/M_Character_Teeth",
  "Shaders": [
    {
      "ShaderFile": "shaders/005_unknown_idx_46518_group_02843.dxil",
      "ShaderIndex": 46518,
      "Stage": "Pixel Shader",
      "ShaderModel": "ps_6_6",
      "RoleCandidate": "BasePassPixelCandidate",
      "VariantKind": "surface_candidate|supporting_vertex|runtime_only|unknown",
      "FeatureCounts": {
        "InstructionCount": 0,
        "TextureOperations": 0,
        "TextureRegisters": 0,
        "CBufferLoads": 0,
        "UavOperations": 0,
        "MrtOutputs": 0,
        "RuntimeFeatureHints": 0
      },
      "ResourceSignature": {
        "TextureRegisters": ["t0", "t1"],
        "SamplerRegisters": ["s0"],
        "CBufferRegisters": ["cb0"],
        "UavRegisters": []
      },
      "OutputSignature": {
        "Targets": ["SV_Target0", "SV_Target1"],
        "MrtCount": 2
      },
      "RuntimeFeatures": [],
      "Scores": {
        "SurfaceRelevance": 0.0,
        "Completeness": 0.0,
        "RuntimePenalty": 0.0,
        "FinalSelectionScore": 0.0
      },
      "Evidence": [],
      "SourceFiles": []
    }
  ]
}
```

Feature extraction rules:

```text
TextureOperations = dxil_resource_usage.TextureOperations.Count
TextureRegisters = distinct TextureOperations.Register
CBufferLoads = dxil_resource_usage.CBufferLoads.Count
UavOperations = dxil_resource_usage.UavOperations.Count
MrtOutputs = distinct pixel shader Outputs.RowIndex
RuntimeFeatureHints = runtime_features references + DXIL metadata hints
InstructionCount = dxil_resource_usage.InstructionCount
```

## Phase 2 - Scoring

新增 `ShaderReconstructionScorer`。

Scores:

```text
SurfaceRelevance
Completeness
RuntimePenalty
RedundancyPenalty
FinalSelectionScore
```

Suggested scoring:

```text
Pixel Shader base:
  +0.40 if Stage == Pixel Shader
  +0.20 if RoleCandidate contains BasePass/Pixel/Material
  +0.10 if MRT outputs > 0
  +0.10 if texture operations > 0
  +0.10 if texture registers overlap semantic_binding_map candidates
  +0.05 if cbuffer loads > 0

Vertex Shader base:
  +0.25 if Stage == Vertex Shader
  +0.10 if used by same group or similar resource signature
  +0.10 if has world/uv/normal input/output signature hints

Penalty:
  -0.35 if UavOperations > 0
  -0.30 if runtime_features contains VirtualTextureFeedbackUAV
  -0.25 if Stage == Compute Shader for material reconstruction
  -0.20 if MetadataHints contains PageTable/Feedback/Nanite/Shadow
```

Classification:

```text
surface_candidate:
  Pixel Shader with high surface relevance and not runtime-only.

supporting_vertex:
  Vertex Shader with useful input/output signature or matching group.

runtime_only:
  Compute Shader, UAV-heavy shader, VT feedback/page-table shader, renderer-only shader.

unknown:
  insufficient evidence.
```

Every score must include reason strings.

## Phase 3 - Similarity Grouping

新增 `ShaderSimilarityGrouper`。

Purpose:

```text
把多份差异很小的 permutation 合并成 group，让 AI 只读 group representative。
```

Similarity features:

```text
Stage
RoleCandidate
Texture register set
Sampler register set
CBuffer register set
UAV register set
MRT target set
Texture operation count bucket
CBuffer load count bucket
Runtime feature flags
Shader model
```

Exact signature:

```text
stage|role|textures|samplers|cbuffers|uavs|targets|runtimeFlags|shaderModel
```

Fuzzy similarity:

```text
Jaccard(TextureRegisters) >= 0.80
Jaccard(CBufferRegisters) >= 0.80
abs(TextureOpsA - TextureOpsB) <= 20%
abs(CBufferLoadsA - CBufferLoadsB) <= 20%
same Stage
same runtime-only classification
```

Output:

```text
analysis/shader_similarity_groups.json
```

Schema:

```json
{
  "Groups": [
    {
      "GroupId": "ps_surface_0",
      "RepresentativeShaderFile": "shaders/005_unknown_idx_46518_group_02843.dxil",
      "Stage": "Pixel Shader",
      "VariantKind": "surface_candidate",
      "Members": [],
      "SimilarityBasis": [],
      "RepresentativeReason": "",
      "Confidence": 0.0,
      "Evidence": []
    }
  ]
}
```

Representative selection:

```text
highest FinalSelectionScore
then highest TextureOperations
then highest MrtOutputs
then highest CBufferLoads
then lowest RuntimePenalty
```

## Phase 4 - Reconstruction Entrypoints

新增 `ReconstructionEntrypointSelector`。

Output:

```text
analysis/reconstruction_entrypoints.json
```

Schema:

```json
{
  "Material": "/Game/Materials/_Master/Master/M_Character_Teeth",
  "Strategy": "Use primary pixel shader representatives for surface/deferred reconstruction; use vertex shaders only as input evidence; abstract runtime-only shaders.",
  "PrimaryPixelShaders": [
    {
      "ShaderFile": "",
      "GroupId": "",
      "UseFor": "surface_and_deferred_output_reconstruction",
      "Score": 0.0,
      "Reason": "",
      "Evidence": [],
      "SourceFiles": []
    }
  ],
  "SupportingVertexShaders": [
    {
      "ShaderFile": "",
      "UseFor": "uv_world_position_normal_input_evidence",
      "Score": 0.0,
      "Reason": "",
      "Evidence": [],
      "SourceFiles": []
    }
  ],
  "RuntimeOnlyOrAbstract": [
    {
      "ShaderFile": "",
      "Reason": "Virtual texture feedback UAV / compute-only / renderer runtime feature",
      "UnityHandling": "abstract",
      "Evidence": [],
      "SourceFiles": []
    }
  ],
  "IgnoredSimilarVariants": [
    {
      "ShaderFile": "",
      "RepresentativeShaderFile": "",
      "SimilarityGroup": "",
      "Reason": ""
    }
  ],
  "KnownRisks": []
}
```

Selection defaults:

```text
PrimaryPixelShaders max: 2
SupportingVertexShaders max: 2
RuntimeOnlyOrAbstract: unlimited, but summarized
IgnoredSimilarVariants: all non-representative group members
```

## Phase 5 - Variant Diff Summary

新增 `VariantDiffSummarizer`。

Purpose:

```text
给 AI Agent 一个“为什么这些变体可以不全读”的解释。
```

Output:

```text
analysis/variant_diff_summary.json
```

Schema:

```json
{
  "Summary": {
    "TotalShaders": 14,
    "SimilarityGroups": 0,
    "PrimaryPixelShaderCount": 0,
    "RuntimeOnlyCount": 0,
    "IgnoredSimilarVariantCount": 0
  },
  "Groups": [
    {
      "GroupId": "",
      "Representative": "",
      "MemberCount": 0,
      "CommonFeatures": [],
      "Differences": [],
      "Recommendation": "read_representative_only|read_all|abstract"
    }
  ]
}
```

## Phase 6 - Deferred Lighting Reuse Policy

新增 `DeferredLightingPolicyWriter`。

Purpose:

```text
明确 Unity Deferred 还原边界：还原 Material/BasePass -> surface/GBuffer 逻辑，
不把 UE Deferred Light Pass 搬进普通 Unity material shader。
```

Output:

```text
analysis/deferred_lighting_policy.json
```

Schema:

```json
{
  "LightingPassStrategy": "reuse_unity_urp_deferred_lighting",
  "ReconstructScope": [
    "texture_sampling",
    "material_parameter_application",
    "normal_reconstruction",
    "surface_attribute_calculation",
    "gbuffer_or_deferred_output_mapping"
  ],
  "DoNotReconstruct": [
    "ue_deferred_light_accumulation",
    "ue_shadow_or_light_grid_code",
    "ue_reflection_environment_lighting",
    "ue_renderer_only_virtual_texture_feedback",
    "ue_renderer_only_page_table_logic"
  ],
  "UnityStrategy": {
    "Default": "Use Unity URP Deferred lighting; reconstruct only material surface inputs.",
    "IfCustomLightPassDetected": "Do not port into material shader; emit RendererFeature/fullscreen-pass/custom-BRDF requirement.",
    "IfSubsurfaceDetected": "Document approximation or custom pass requirement."
  },
  "Risks": [
    "UE and Unity BRDF are not bit-equivalent.",
    "MSM_Subsurface may not map directly to URP Deferred.",
    "CustomData/GBuffer channels may not map 1:1 to URP Deferred."
  ],
  "Evidence": [],
  "SourceFiles": []
}
```

Rules:

```text
Default policy is always reuse_unity_urp_deferred_lighting.
Material shader reconstruction must not require reading UE global LightPass shaders.
If LightPass audit detects custom renderer lighting, output a custom lighting requirement instead of expanding material shader scope.
```

## Phase 7 - LightPass Candidate Scan

新增 `LightPassCandidateScanner`。

Purpose:

```text
轻量审计 shader archive / .stinfo / shader metadata 中是否存在 Deferred LightPass / global lighting shader，
并判断是否有 project/plugin custom shader 痕迹。
```

Inputs:

```text
source/shader_type_info/*.stinfo
analysis/shader_type_info.json
source/shader_archive.metadata.json
manifest.json
```

Output:

```text
analysis/lightpass_candidates.json
```

Search terms:

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
BasePass
```

Source path classification:

```text
stock_engine:
  Engine/Shaders/Private/*
  Engine/Shaders/Public/*

project_custom:
  Project/Shaders/*
  /Game/Shaders/*
  Source/*/Shaders/*

plugin_custom:
  Plugins/*/Shaders/*
```

Schema:

```json
{
  "Candidates": [
    {
      "NameOrString": "",
      "SourcePathCandidate": "",
      "CandidateKind": "deferred_light|reflection|shadow|subsurface|gbuffer|unknown",
      "Origin": "stock_engine|project_custom|plugin_custom|unknown",
      "Confidence": 0.0,
      "Evidence": [],
      "SourceFiles": []
    }
  ],
  "CustomShaderSignals": [],
  "StockShaderSignals": []
}
```

## Phase 8 - LightPass Audit

新增 `LightPassAuditor`。

Purpose:

```text
判断原游戏 LightPass 是否 stock-like，还是存在 project/plugin custom renderer lighting 风险。
这个审计只影响 Unity lighting strategy，不扩大 material shader 还原范围。
```

Inputs:

```text
analysis/lightpass_candidates.json
analysis/runtime_features.json
analysis/gbuffer_semantics.json
analysis/dxil_resource_usage.json
analysis/shader_type_info.json
```

Output:

```text
analysis/lightpass_audit.json
```

Audit levels:

```text
Level 1 - name/source scan:
  shader type names, readable .stinfo strings, source path candidates

Level 2 - structural hints:
  GBuffer / SceneTexture reads
  light data / shadow data bindings
  light accumulation output patterns
  BRDF / attenuation / reflection terms in readable strings or metadata hints

Level 3 - optional stock UE comparison:
  compare against known UE 5.6.1 stock shader names/signatures if available
```

Schema:

```json
{
  "Conclusion": "stock_like|custom_lighting_detected|inconclusive",
  "RecommendedUnityStrategy": "reuse_unity_urp_deferred_lighting|reuse_plus_custom_renderer_feature|requires_manual_renderer_review",
  "Confidence": 0.0,
  "Evidence": [],
  "StockLikeSignals": [],
  "CustomSignals": [],
  "RendererOnlyShaders": [],
  "DoNotPortIntoMaterialShader": [],
  "CustomLightingRequirement": {
    "Required": false,
    "SuggestedUnityApproach": "",
    "Reason": ""
  },
  "KnownRisks": []
}
```

Decision rules:

```text
If only Engine/Shaders/Private-style lighting strings are found:
  Conclusion = stock_like or inconclusive with stock-like bias.

If Project/Shaders or Plugins/*/Shaders lighting strings are found:
  Conclusion = custom_lighting_detected.

If no reliable strings are available:
  Conclusion = inconclusive.

Even when custom lighting is detected:
  Do not port LightPass into material shader.
  Emit RendererFeature/fullscreen deferred correction/custom lighting requirement.
```

For `MSM_Subsurface`:

```text
Stock UE LightPass does not imply URP Deferred equivalence.
Subsurface must remain a documented approximation or custom-pass requirement.
```

## Phase 9 - AI Context Pack JSON

新增 `AiContextPackWriter`。

Output:

```text
analysis/ai_context_pack.json
```

Purpose:

```text
给后续 Unity Agent 一个上下文受控、直接可消费的主输入。
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

Schema:

```json
{
  "Target": {
    "UnityVersion": "Unity 6",
    "RenderPipeline": "URP",
    "RenderingPath": "Deferred",
    "DotsInstancing": true
  },
  "Material": {},
  "PrimaryInputs": {
    "Contract": "analysis/unity_deferred_reconstruction_contract.json",
    "SemanticBindingMap": "analysis/semantic_binding_map.json",
    "Entrypoints": "analysis/reconstruction_entrypoints.json"
  },
  "RecommendedReadOrder": [],
  "SelectedShaders": [],
  "TextureBindings": [],
  "UniformBindings": [],
  "RuntimeFeaturesToAbstract": [],
  "DeferredLightingPolicy": {},
  "LightPassAudit": {},
  "GBufferCandidates": [],
  "MaterialFunctionContext": [],
  "KnownUnknowns": [],
  "DoNotReadByDefault": [],
  "EvidenceDrillDown": []
}
```

Important rule:

```text
ai_context_pack.json must not inline full .dxil.ll files.
It may include short line references or small snippets only when needed.
```

## Phase 10 - AI Context Pack Markdown

Output:

```text
analysis/ai_context_pack.md
```

Purpose:

```text
让新 AI Agent 即使不先解析 JSON，也能快速知道怎么开始。
```

Sections:

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

## Phase 11 - CLI Integration

新增 CLI:

```text
--analyze-variants
--write-ai-context-pack
--ai-context-budget <characters>
--max-primary-variants <count>
--max-supporting-variants <count>
--context-only <existing bundle>
--audit-lightpass
```

Behavior:

```text
Default single export:
  semantic analysis enabled
  variant analysis enabled
  ai context pack enabled

--no-semantic-analysis:
  disables semantic and dependent AI context outputs

--semantic-only:
  should also be able to regenerate variant analysis and AI context pack from existing semantic JSON

--context-only:
  only regenerates variant reduction and AI context files from existing bundle

--audit-lightpass:
  enables deferred lighting policy and lightpass audit outputs
```

`--context-only` does not require paks, mapping, dxc, or decompress_shader.

## Phase 12 - Verify Integration

Extend `--verify-only`:

```text
analysis/variant_inventory.json parseable
analysis/shader_similarity_groups.json parseable
analysis/reconstruction_entrypoints.json parseable
analysis/variant_diff_summary.json parseable
analysis/deferred_lighting_policy.json parseable
analysis/lightpass_candidates.json parseable
analysis/lightpass_audit.json parseable
analysis/ai_context_pack.json parseable
analysis/ai_context_pack.md exists
all selected ShaderFile paths exist
all ignored variants point to existing representative
all confidence/score values are 0..1
primary pixel shader count <= configured max
ai_context_pack.json does not inline full .dxil.ll text
lightpass audit never instructs material shader to port UE LightPass directly
```

## Phase 13 - ShaderWP Agent Docs

Update:

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
9. drill down only when needed
```

## Phase 14 - Per-Export Agent Workspace Docs

Agent handoff docs must be generated by the exporter itself, not copied by hand after a golden-case export.

Every single-bundle output directory must contain:

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

Exporter integration:

```text
full export:
  write Agent workspace after manifest generation
  update manifest AgentWorkspace section
  refresh manifest Files list after docs/scripts are written

--context-only:
  regenerate AI context
  repair Agent workspace docs/scripts for existing bundle
  verify bundle

--semantic-only:
  regenerate semantic outputs
  regenerate AI context
  repair Agent workspace docs/scripts
  verify bundle

batch export:
  each child bundle writes its own Agent workspace
  batch root writes summary-level AGENTS/WORKFLOW/NEXT_TASK/PROMPT/agent_context and verify_all script
```

Verification:

```text
--verify-only requires root Agent workspace files
agent_context.json Schema == ue-cooked-shader-agent-workspace/v1
agent_context.json Target.RenderingPath == Deferred
agent_context.json Target.DotsInstancing == true
agent_context.json ReadFirst includes analysis/ai_context_pack.md
manifest AgentWorkspace.Target.RenderingPath == Deferred
```

This phase prevents a fresh AI Agent from starting with raw `shaders/*.dxil.ll` or missing the Unity 6 URP Deferred + DOTS reconstruction contract.

## Golden Case Acceptance

For `M_Character_Teeth.bundle`, expected result:

```text
verify-only returns Verify: OK
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
reconstruction_entrypoints PrimaryPixelShaders count >= 1
reconstruction_entrypoints PrimaryPixelShaders count <= 2
reconstruction_entrypoints RuntimeOnlyOrAbstract count >= 1
ai_context_pack Target.RenderingPath == Deferred
ai_context_pack Target.DotsInstancing == true
ai_context_pack DeferredLightingPolicy.LightingPassStrategy == reuse_unity_urp_deferred_lighting
lightpass_audit RecommendedUnityStrategy exists
ai_context_pack DoNotReadByDefault includes non-representative variants or raw .dxil.ll files
```

## Implementation Notes

The first version can be heuristic, but it must be deterministic:

```text
same inputs -> same selected representatives
all scores explainable
no material-specific filename hardcoding
all outputs bundle-relative
```

If a shader cannot be confidently classified, classify it as `unknown` and include it in `EvidenceDrillDown`, not as a primary shader.
