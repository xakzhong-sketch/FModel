# UE Module-first / MI-driven Unity Reconstruction Tasks

本文档把 `Doc/UE_Module_First_MI_Driven_Unity_Reconstruction_Plan.md` 拆解成可执行任务。

目标：

```text
让 CUE4Parse.ShaderBundleExporter 能输出全局 Master/Layer/Blend 模块认知、MI layer-aware contract、shader reuse 判断，并让 Unity 还原流程按可复用模块库 + 结构化 shader 复用执行。
```

## Task Status Legend

```text
todo       尚未开始
doing      正在实现
blocked    需要外部信息或前置任务
done       已完成并验证
```

## Phase 0: Schema and Fixtures

### T0.1 Define JSON Schemas

Status: todo

Deliverables:

```text
analysis/material_family.json
analysis/material_layer_stack.json
analysis/material_layer_parameter_bindings.json
analysis/material_static_permutation.json
analysis/unity_shader_reuse_key.json
analysis/unity_shader_reuse_candidates.json
analysis/unity_shader_assignment.json
analysis/unity_layer_reconstruction_contract.json
```

Implementation notes:

```text
1. Use StableKey = Association + Index + Name.
2. Every inferred field needs Confidence and Evidence.
3. Do not include ordinary parameter values in shader reuse key.
4. Include SourceFiles for every analyzer output.
```

Acceptance:

```text
MI_CG_RockSmooth_01a can be represented without merging duplicate parameter names.
```

### T0.2 Add Fixture Expectations

Status: todo

Fixture:

```text
D:\Tmp\ShaderBundles\MI_CG_RockSmooth_01a.bundle
```

Expected stack:

```text
Layers:
  MLI_CG_RockSmooth_01a
  MLI_CG_RockSmooth_01b
  ML_LayerTint
  ML_LayerTint
  ML_LayerTint
  ML_LayerCustomPrim_CurveGradient

Blends:
  MB_MaskID
  MB_VertexColorOverlay
  MB_VertexColorOverlay
  MB_VertexColorOverlay
  MB_CustomPrim_Overlay
```

Acceptance:

```text
Fixture checks can fail clearly when a parser regresses.
```

## Phase 1: Project-level Module Catalog

### T1.1 Implement MasterMaterialCatalogExporter

Status: todo

Scope:

```text
/Game/Materials/_Master/Master/M_*
```

Output:

```text
analysis/master_material_catalog.json
source/master_materials/*.cooked.json
```

Acceptance:

```text
Catalog includes M_LayerStandard and M_Trimsheet_LayerStandard when present in mounted VFS.
Each entry records ObjectPath, asset type, export status, parent/dependency summary, and source file.
```

### T1.2 Implement MaterialLayerModuleCatalogExporter

Status: todo

Scope:

```text
/Game/Materials/_Master/Layers/ML_*
```

Output:

```text
analysis/material_layer_module_catalog.json
source/material_layers/*.cooked.json
```

Acceptance:

```text
Catalog includes known modules such as ML_LayerStandard, ML_LayerTint, ML_LayerCustomPrim_CurveGradient, ML_TrimSheet.
```

### T1.3 Implement MaterialBlendModuleCatalogExporter

Status: todo

Scope:

```text
/Game/Materials/_Master/Blends/MB_*
```

Output:

```text
analysis/material_blend_module_catalog.json
source/material_blends/*.cooked.json
```

Acceptance:

```text
Catalog includes known modules such as MB_MaskID, MB_VertexColorOverlay, MB_CustomPrim_Overlay, MB_HeightLerp, MB_TrimSheet.
```

### T1.4 Add Catalog CLI Mode

Status: todo

CLI:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --game Subnautica2 `
  --paks "..." `
  --mapping "..." `
  --export-master-material-catalog `
  --out "D:\ShaderWP\Subnautica2_ModuleCatalog" `
  --verbose
```

Acceptance:

```text
Command writes catalog root without requiring a single --material input.
```

## Phase 2: Material Family Usage Stats

### T2.1 Implement MaterialFamilyUsageStatsExporter

Status: todo

Output:

```text
analysis/master_material_usage_stats.json
analysis/material_layer_usage_stats.json
analysis/material_blend_usage_stats.json
analysis/material_family_reconstruction_priority.json
```

Acceptance:

```text
Can rank M_LayerStandard materials by common Layer/Blend combinations.
Can identify representative MIs for environment, trimsheet, hard-surface, character/creature, and VFX/translucent groups.
```

### T2.2 Add Usage Scan CLI Mode

Status: todo

CLI:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --game Subnautica2 `
  --paks "..." `
  --mapping "..." `
  --scan-material-family-usage `
  --parent "/Game/Materials/_Master/Master/M_LayerStandard" `
  --out "D:\ShaderWP\Subnautica2_M_LayerStandard_Usage" `
  --verbose
```

Acceptance:

```text
Scan produces deterministic JSON and useful summary warnings for unresolved materials.
```

## Phase 3: MI Layer-aware Bundle Analysis

### T3.1 Implement MaterialFamilyAnalyzer

Status: todo

Output:

```text
analysis/material_family.json
```

Fields:

```text
Material path
Parent path/name
HasStaticPermutationResource
HasMaterialLayers
BlendMode
Render-state hints
Family confidence and evidence
```

Acceptance:

```text
MI_CG_RockSmooth_01a reports parent M_LayerStandard and HasMaterialLayers=true.
```

### T3.2 Implement MaterialLayerStackAnalyzer

Status: todo

Output:

```text
analysis/material_layer_stack.json
```

Requirements:

```text
1. Extract ordered Layers and Blends.
2. Preserve duplicate module occurrences.
3. Preserve Layer/Blend Index.
4. Record ObjectName, ObjectPath, Kind, SourcePath, Confidence.
```

Acceptance:

```text
MI_CG_RockSmooth_01a stack matches the fixture list exactly.
```

### T3.3 Implement MaterialLayerAssetExporter

Status: todo

Output:

```text
source/material_layers/*.cooked.json
source/material_layer_instances/*.cooked.json
source/material_blends/*.cooked.json
```

Requirements:

```text
1. Export referenced ML_*, MB_*, MLI_* assets from active stack.
2. Try to resolve MLI parent layer.
3. Missing referenced assets should create warnings, not hard-fail the whole export.
```

Acceptance:

```text
MI_CG_RockSmooth_01a bundle contains cooked JSON for referenced _Master layers/blends and local MLI assets when available.
```

### T3.4 Implement LayerAwareParameterBindingAnalyzer

Status: todo

Output:

```text
analysis/material_layer_parameter_bindings.json
```

Requirements:

```text
1. Build StableKey = Association + Index + Name.
2. Merge Scalar/Vector/Texture parameters.
3. Prefer value sources in this order:
   MaterialInstanceOverride > StaticPermutation > UniformExpressionSetFinal > ParentDefault.
4. Generate deterministic UnityName.
5. Report collisions explicitly.
```

Acceptance:

```text
Duplicate names like Vertex Paint - Opacity do not collide.
Unity property names are collision-free and stable across repeated exports.
```

### T3.5 Implement MaterialStaticPermutationAnalyzer

Status: todo

Output:

```text
analysis/material_static_permutation.json
```

Requirements:

```text
1. Extract direct static switches when exposed by CUE4Parse.
2. Infer active features from FunctionInfos and active Layer/Blend stack when direct data is missing.
3. Keep direct facts separate from inferred hints.
```

Acceptance:

```text
NormalFromHeightmap, MaskID, VertexColorOverlay, CustomPrimCurveGradient can appear as active feature hints for MI_CG_RockSmooth_01a with evidence.
```

## Phase 4: Shader Reuse Analysis

### T4.1 Implement UnityShaderRegistryReader

Status: todo

Input:

```text
Assets/Shaders/Subnautica2/MaterialModules/Registry/unity_shader_registry.json
```

Acceptance:

```text
Analyzer can operate when registry is missing, empty, or populated.
Missing registry means no existing shader candidates.
```

### T4.2 Implement UnityShaderReuseKeyWriter

Status: todo

Output:

```text
analysis/unity_shader_reuse_key.json
```

Key inputs:

```text
Parent
BlendMode/render-state hints
Layer module sequence
Blend module sequence
StaticFeatureSet
RequiredModuleBranches
PropertySchemaHash
ModuleBranchHash
```

Do not include:

```text
ordinary scalar/vector values
ordinary texture asset choices
material instance display name
```

Acceptance:

```text
Two MIs with identical structure but different parameter values produce the same reuse key hash.
```

### T4.3 Implement UnityShaderReuseCandidateAnalyzer

Status: todo

Output:

```text
analysis/unity_shader_reuse_candidates.json
```

Candidate kinds:

```text
exact
extendable
incompatible
none
```

Acceptance:

```text
Candidate report explains why a shader can be reused, must be extended, or cannot be used.
```

### T4.4 Implement UnityShaderAssignmentWriter

Status: todo

Output:

```text
analysis/unity_shader_assignment.json
```

Decisions:

```text
reuse_existing
extend_existing
create_new
```

Acceptance:

```text
Fresh Agent can decide whether to write shader code by reading assignment JSON.
```

## Phase 5: Unity Layer Reconstruction Contract

### T5.1 Implement UnityLayerContractWriter

Status: todo

Output:

```text
analysis/unity_layer_reconstruction_contract.json
```

Inputs:

```text
material_family.json
material_layer_stack.json
material_layer_parameter_bindings.json
material_static_permutation.json
unity_shader_reuse_key.json
unity_shader_assignment.json
unity_deferred_reconstruction_contract.json
semantic_binding_map.json
```

Acceptance:

```text
Contract lists required modules, required branches, StableKey properties, shader assignment, and unsupported runtime features.
```

## Phase 6: Agent Workspace and Verify Updates

### T6.1 Update Generated Agent Docs

Status: todo

Files:

```text
README.md
AGENTS.md
WORKFLOW.md
NEXT_TASK.md
PROMPT_NEXT_SESSION.md
skills/unity6-urp-deferred-shader-reconstruction/SKILL.md
```

New rules:

```text
1. Read unity_layer_reconstruction_contract.json before raw DXIL.
2. Read shader reuse key/candidates/assignment before writing shader code.
3. Reuse existing Unity Shader when assignment says reuse_existing.
4. Do not merge same-name parameters across Association/Index.
5. Parameter differences create different .mat files, not different shaders.
```

Acceptance:

```text
Fresh Agent can follow generated docs without chat history.
```

### T6.2 Update agent_context.json

Status: todo

Add:

```text
Layer-aware files
Shader reuse files
Module-first / MI-driven strategy
Registry path hint
```

Acceptance:

```text
agent_context.json is parseable and includes layer-aware read order.
```

### T6.3 Update verify_bundle.ps1 / --verify-only

Status: todo

Rules:

```text
If HasMaterialLayers=true, verify required layer-aware files.
If shader assignment exists, verify it is parseable.
Initial implementation may warn instead of hard fail until analyzer is stable.
```

Acceptance:

```text
--verify-only reports missing layer-aware contract files clearly.
```

## Phase 7: Unity Material Restore Upgrade

### T7.1 Add --layer-aware Mode

Status: todo

Script:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Tools/unity_material_apply_ue_params.py
```

Behavior:

```text
1. Read analysis/material_layer_parameter_bindings.json.
2. Match StableKey -> UnityName first.
3. Fall back to legacy name matching only if layer-aware map is missing.
4. Report fallback usage.
```

Acceptance:

```text
Can dry-run against a layer-aware Unity shader and report MatchedByStableKey.
```

### T7.2 Add Collision and Schema Reports

Status: todo

Report fields:

```text
LayerAware
MatchedByStableKey
MatchedByLegacyNameFallback
Collisions
SkippedMissingUnityProperties
MissingTextureGuids
```

Acceptance:

```text
Duplicate UE display names do not silently overwrite Unity material values.
```

### T7.3 Support Shader Assignment Awareness

Status: todo

Behavior:

```text
If unity_shader_assignment.json.Decision=reuse_existing, report AssignedUnityShader and do not expect a MI-specific shader name.
```

Acceptance:

```text
Material restore report states which shared shader the .mat should use.
```

## Phase 8: First End-to-end Target

### T8.1 Export MI_CG_RockSmooth_01a Layer-aware Bundle

Status: todo

Command shape:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --game Subnautica2 `
  --paks "D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\Subnautica2\Content\Paks" `
  --mapping "D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\5.6.1-114707+++Project+SN2-Release-Hotfix-Live-Subnautica2.usmap" `
  --material "/Game/Art/Surfaces/Tiling/CoralGarden/Rock/CG_RockSmooth_01a/MI_CG_RockSmooth_01a" `
  --out "K:\WorkSpace\ShaderReverse\MI_CG_RockSmooth_01a\MI_CG_RockSmooth_01a.bundle" `
  --include-layer-stack `
  --include-master-modules `
  --decompress-shader "D:\Github\UEShaderMapExtractor\Build\decompress_shader.exe" `
  --overwrite `
  --verbose
```

Acceptance:

```text
Verify: OK
Layer stack matches fixture.
Unity shader assignment is present.
```

### T8.2 Build First Reusable Module Set

Status: todo

Expected modules:

```text
MaterialModules/Master/M_LayerStandard_Common.hlsl
MaterialModules/Layers/ML_LayerStandard.hlsl
MaterialModules/Layers/ML_LayerTint.hlsl
MaterialModules/Layers/ML_LayerCustomPrim_CurveGradient.hlsl
MaterialModules/Blends/MB_MaskID.hlsl
MaterialModules/Blends/MB_VertexColorOverlay.hlsl
MaterialModules/Blends/MB_CustomPrim_Overlay.hlsl
MaterialModules/Runtime/SN2MaterialAttributes.hlsl
```

Acceptance:

```text
Each module has .module.json.
Each module lists implemented and unimplemented branches.
```

### T8.3 Compose First Shader or Reuse Existing

Status: todo

Rules:

```text
If assignment=create_new, create structural shader name such as LayerStack_*.shader.
If assignment=reuse_existing, do not write shader code.
```

Acceptance:

```text
Unity shader compiles in Unity 6 URP Deferred and supports DOTS instancing.
```

### T8.4 Restore MI_CG_RockSmooth_01a .mat

Status: todo

Use:

```powershell
python D:\Github\FModel\CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_material_apply_ue_params.py `
  --mat "K:\...\MI_CG_RockSmooth_01a.mat" `
  --bundle-dir "K:\WorkSpace\ShaderReverse\MI_CG_RockSmooth_01a\MI_CG_RockSmooth_01a.bundle" `
  --layer-aware `
  --report-out "K:\WorkSpace\ShaderReverse\MI_CG_RockSmooth_01a\unity_material_restore_report.json"
```

Acceptance:

```text
MatchedByStableKey covers core + layer + blend parameters.
Collisions is empty.
MissingTextureGuids is empty or documented.
```

## Phase 9: Reuse Validation with Second MI

### T9.1 Select Similar MI

Status: todo

Criteria:

```text
Same parent M_LayerStandard.
Same or highly similar Layer/Blend module sequence.
Different textures/parameter values.
```

Acceptance:

```text
Selected MI should test shader reuse instead of new shader creation.
```

### T9.2 Export Second MI Bundle and Compare Reuse Key

Status: todo

Acceptance:

```text
If structure matches, unity_shader_assignment.json says reuse_existing.
If structure differs only by missing branch, assignment says extend_existing.
```

### T9.3 Restore Second .mat Without New Shader

Status: todo

Acceptance:

```text
Second Unity .mat uses the existing shared shader and has different restored parameters/textures.
Shader registry records both validated MIs.
```

## Phase 10: Regression and Quality Gates

### T10.1 Build Verification

Status: todo

Commands:

```powershell
dotnet build CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release
python -m py_compile CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_material_apply_ue_params.py
```

Acceptance:

```text
Build succeeds.
Python script compiles.
Known cmake/native skip warning remains non-blocking if unchanged.
```

### T10.2 JSON Parse Verification

Status: todo

Scope:

```text
All new analysis/*.json files.
agent_context.json.
module metadata JSON.
unity_shader_registry.json.
```

Acceptance:

```text
All JSON files parse with no trailing-text or encoding errors.
```

### T10.3 Agent Handoff Verification

Status: todo

Acceptance:

```text
A fresh Agent can start from PROMPT_NEXT_SESSION.md and know:
  read layer contract first
  evaluate shader reuse
  reuse modules
  create/restore .mat when shader is shared
  use StableKey properties
```

## Phase 11: Documentation Updates

### T11.1 Update Goal Docs

Status: todo

Files:

```text
Doc/UE_Cooked_Material_Bundle_Export_Goal.md
Doc/UE_Unity_Material_Property_Restore_Goal.md
```

Acceptance:

```text
Goal docs mention layer-aware export, shader reuse assignment, and StableKey material restore.
```

### T11.2 Add Reconstruction Goal for Module-first Flow

Status: todo

New file:

```text
Doc/UE_Module_First_MI_Driven_Unity_Reconstruction_Goal.md
```

Acceptance:

```text
New session can invoke the goal and follow module-first / shader-reuse workflow directly.
```

## Overall Done Criteria

The task set is complete when:

```text
1. Project-level module catalog export works.
2. M_LayerStandard usage stats export works.
3. MI_CG_RockSmooth_01a layer-aware bundle exports and verifies.
4. Layer stack, parameter bindings, static permutation, shader reuse, and Unity layer contract files exist.
5. Unity material restore supports --layer-aware and StableKey matching.
6. A first Unity shader/module set is created or assigned from registry.
7. A second structurally compatible MI reuses the existing Unity Shader and only creates/restores a new .mat.
8. Generated Agent docs describe the full workflow without relying on chat history.
```
