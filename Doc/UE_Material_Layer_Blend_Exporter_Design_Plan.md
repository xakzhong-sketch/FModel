# UE Material Layer / Blend Exporter Design Plan

本文档定义 `CUE4Parse.ShaderBundleExporter` 的下一阶段调整：把当前 cooked shader bundle 从“单材质 shader 证据包”升级为“Material family + MI static permutation + Unity layer-aware reconstruction contract”。

目标是解决 `M_LayerStandard` / `M_Trimsheet_LayerStandard` 这类 UE Material Layers 体系在 Unity 还原中的核心问题：

```text
父材质默认 bundle 只能覆盖核心 surface path。
具体 MI 可能有自己的 static permutation、Material Layer stack、Layer Instance、Blend stack、参数覆盖和函数依赖。
如果 Unity Shader 只按父材质核心参数还原，会缺失 Layer/Blend/VertexPaint/Gradient/Mask/NormalFromHeightmap 等路径。
```

## Problem Statement

以 `MI_CG_RockSmooth_01a` 为例：

```text
Parent:
  /Game/Materials/_Master/Master/M_LayerStandard

MI facts:
  bHasStaticPermutationResource = true
  bHasMaterialLayers = true

Active stack:
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

当前 Unity shader 还原基于 `M_LayerStandard.bundle` 的父材质默认/核心路径，已经能覆盖 BCM/NRH、BaseColor、Normal、Roughness、Metallic/Specular、UV 等主路径，但没有完整纳入这个 MI active stack 中的：

```text
ML_LayerTint
ML_LayerCustomPrim_CurveGradient
MB_MaskID
MB_VertexColorOverlay
MB_CustomPrim_Overlay
NormalFromHeightmap
HeightLerp
Vertex Paint blend
Gradient/mask/color-id controls
```

所以缺参数不是简单的 `.mat` 回填问题，而是 exporter 没有把 Material Layer / Blend 语义整理成 Unity Agent 可消费的输入。

## Core Principle

需要把 UE 材质拆成两层：

```text
1. Material family module library
   来自 /Game/Materials/_Master/Master、/Layers、/Blends。
   用来还原可复用模块。

2. Material instance active permutation
   来自具体 MI cooked JSON / shader map / static permutation。
   用来确定某个材质实际启用了哪些模块、顺序、参数覆盖和 shader evidence。
```

不能只做 `_Master` 模块目录，也不能只做几个典型 MI。两者职责不同：

```text
_Master catalog:
  解决有哪些模块、模块定义是什么、模块间函数依赖是什么。

MI bundle:
  解决哪些模块实际被使用、Layer/Blend index 是什么、哪些 static branches 被编译、参数最终值是什么。

Usage stats:
  解决优先模块化哪些 ML/MB/Master 最划算。
```

## Target Outputs

### Bundle-local Outputs

每个单材质 bundle 新增：

```text
source/master_materials/*.cooked.json
source/material_layers/*.cooked.json
source/material_layer_instances/*.cooked.json
source/material_blends/*.cooked.json

analysis/material_family.json
analysis/material_layer_stack.json
analysis/material_layer_parameter_bindings.json
analysis/material_static_permutation.json
analysis/material_layer_function_dependencies.json
analysis/unity_layer_reconstruction_contract.json
```

现有输出继续保留：

```text
parameters/material_parameters.json
parameters/textures.json
analysis/unity_deferred_reconstruction_contract.json
analysis/semantic_binding_map.json
analysis/texture_register_candidates.json
analysis/texture_register_statistics.json
```

### Batch / Project-level Outputs

批量扫描材质或 `_Master` 目录时新增：

```text
analysis/master_material_catalog.json
analysis/material_layer_module_catalog.json
analysis/material_blend_module_catalog.json
analysis/material_layer_instance_catalog.json
analysis/master_material_usage_stats.json
analysis/material_layer_usage_stats.json
analysis/material_blend_usage_stats.json
analysis/material_family_reconstruction_priority.json
```

## Data Model

### Stable Parameter Key

UE Material Layers 参数不能只用 `Name` 匹配。必须使用稳定键：

```text
ParameterName + Association + Index
```

建议 JSON key：

```text
LayerParameter:5:Gradient Curve
BlendParameter:0:Threshold
GlobalParameter:-1:SelectionColor
```

字段：

```json
{
  "StableKey": "BlendParameter:0:Threshold",
  "UEName": "Threshold",
  "Association": "BlendParameter",
  "Index": 0,
  "ParameterType": "Scalar",
  "Value": 0.2,
  "Source": "MaterialInstanceOverride|UniformExpressionSet|ParentDefault",
  "SourcePath": "source/material.cooked.json:ScalarParameterValues",
  "Evidence": [
    "ParameterInfo has Association=BlendParameter and Index=0.",
    "Material instance override value is present."
  ]
}
```

### Unity Property Name

Unity internal property name should be deterministic and collision-free:

```text
_Global_SelectionColor
_Layer5_Gradient_Curve
_Blend0_Threshold
_Layer0_Normal_Intensity
```

Display name can preserve original UE name:

```text
"Gradient Curve"
"Threshold"
```

Mapping output:

```json
{
  "StableKey": "LayerParameter:5:Gradient Curve",
  "UnityName": "_Layer5_Gradient_Curve",
  "UnityDisplayName": "Gradient Curve",
  "UEName": "Gradient Curve",
  "Association": "LayerParameter",
  "Index": 5,
  "Type": "Scalar",
  "DefaultValue": 17.0
}
```

### Material Family

`analysis/material_family.json`:

```json
{
  "Material": "/Game/Art/.../MI_CG_RockSmooth_01a",
  "Parent": "/Game/Materials/_Master/Master/M_LayerStandard",
  "ParentName": "M_LayerStandard",
  "HasStaticPermutationResource": true,
  "HasMaterialLayers": true,
  "Family": "M_LayerStandard",
  "FamilyConfidence": 1.0,
  "Evidence": [
    "MaterialInstanceConstant.Properties.Parent points to M_LayerStandard.",
    "bHasMaterialLayers is true."
  ]
}
```

### Material Layer Stack

`analysis/material_layer_stack.json`:

```json
{
  "Schema": "ue-material-layer-stack/v1",
  "Material": "/Game/Art/.../MI_CG_RockSmooth_01a",
  "Parent": "/Game/Materials/_Master/Master/M_LayerStandard",
  "Layers": [
    {
      "Index": 0,
      "Name": "MLI_CG_RockSmooth_01a",
      "Kind": "MaterialFunctionMaterialLayerInstance",
      "ObjectPath": "/Game/Art/.../MLI_CG_RockSmooth_01a.0",
      "ResolvedParentLayer": "ML_LayerStandard",
      "ResolvedParentLayerPath": "/Game/Materials/_Master/Layers/ML_LayerStandard.0",
      "Confidence": 0.8,
      "Evidence": []
    },
    {
      "Index": 5,
      "Name": "ML_LayerCustomPrim_CurveGradient",
      "Kind": "MaterialFunctionMaterialLayer",
      "ObjectPath": "/Game/Materials/_Master/Layers/ML_LayerCustomPrim_CurveGradient.0",
      "Confidence": 1.0,
      "Evidence": []
    }
  ],
  "Blends": [
    {
      "Index": 0,
      "Name": "MB_MaskID",
      "Kind": "MaterialFunctionMaterialLayerBlend",
      "ObjectPath": "/Game/Materials/_Master/Blends/MB_MaskID.0",
      "Confidence": 1.0,
      "Evidence": []
    }
  ]
}
```

`ResolvedParentLayer` 对 `MLI_*` 可能不是总能确定。必须明确置信度和证据来源。

### Static Permutation

`analysis/material_static_permutation.json`:

```json
{
  "Schema": "ue-material-static-permutation/v1",
  "HasStaticPermutationResource": true,
  "StaticSwitchParameters": [
    {
      "StableKey": "LayerParameter:5:Use Custom Height Texture",
      "UEName": "Use Custom Height Texture",
      "Association": "LayerParameter",
      "Index": 5,
      "Value": true,
      "Source": "StaticParameters|MaterialInstance|CookedGraph",
      "Confidence": 0.7
    }
  ],
  "ActiveFeatureHints": [
    "NormalFromHeightmap",
    "HeightLerp",
    "VertexColorOverlay",
    "MaskID",
    "CustomPrimCurveGradient"
  ],
  "InactiveFeatureHints": []
}
```

如果 cooked JSON 无法直接反序列化 static switch 列表，可从 active layer/blend functions、FunctionInfos、UniformExpressionSet 和 DXIL evidence 推断，标记为 `inferred`。

### Unity Layer Reconstruction Contract

`analysis/unity_layer_reconstruction_contract.json`:

```json
{
  "Schema": "unity-layer-reconstruction-contract/v1",
  "Target": {
    "UnityVersion": "Unity 6",
    "RenderPipeline": "URP",
    "RenderingPath": "Deferred",
    "DotsInstancing": true
  },
  "MaterialFamily": "M_LayerStandard",
  "RecommendedShaderStrategy": "mi_specialized_first_then_module_extract",
  "UnityShaderName": "Subnautica2/Reconstructed/M_LayerStandard/MI_CG_RockSmooth_01a",
  "Modules": [
    {
      "Kind": "Layer",
      "Index": 0,
      "Name": "MLI_CG_RockSmooth_01a",
      "UnityModule": "Layer_MLI_CG_RockSmooth_01a.hlsl",
      "Required": true
    },
    {
      "Kind": "Blend",
      "Index": 0,
      "Name": "MB_MaskID",
      "UnityModule": "Blend_MB_MaskID.hlsl",
      "Required": true
    }
  ],
  "Properties": [],
  "Assumptions": [],
  "BlockedIf": [
    "Source-level UE material graph equivalence is required without accepting cooked evidence candidates.",
    "Layer/Blend static switch state cannot be determined for a required visual feature."
  ]
}
```

## Exporter Architecture Changes

### New Analyzers

Add analyzers under `CUE4Parse.ShaderBundleExporter`:

```text
MaterialFamilyAnalyzer
MasterMaterialCatalogExporter
MaterialLayerStackAnalyzer
MaterialLayerAssetExporter
LayerAwareParameterBindingAnalyzer
MaterialStaticPermutationAnalyzer
UnityLayerContractWriter
MaterialFamilyUsageStatsExporter
```

Suggested responsibilities:

```text
MaterialFamilyAnalyzer
  Read material cooked JSON.
  Resolve Parent material.
  Detect bHasMaterialLayers and bHasStaticPermutationResource.

MasterMaterialCatalogExporter
  Enumerate /Game/Materials/_Master/Master/M_*.
  Enumerate /Game/Materials/_Master/Layers/ML_*.
  Enumerate /Game/Materials/_Master/Blends/MB_*.
  Export cooked JSON and summarize parameters/dependencies.

MaterialLayerStackAnalyzer
  Parse MaterialLayers / LayerNames / FunctionInfos / tree payloads from material cooked JSON.
  Emit active layer/blend stack with index and object path.

MaterialLayerAssetExporter
  Export referenced ML_*, MB_*, and MLI_* assets as cooked JSON.
  Resolve MLI parent layer if available.

LayerAwareParameterBindingAnalyzer
  Build StableKey = Association + Index + Name.
  Merge values from MI overrides, UniformExpressionSet, parent defaults, and texture parameters.
  Emit Unity property names and value source confidence.

MaterialStaticPermutationAnalyzer
  Extract static switches and static component masks where available.
  Infer active features from FunctionInfos and compiled shader evidence when direct data is absent.

UnityLayerContractWriter
  Merge family, stack, parameters, static permutation, semantic binding map, and deferred contract.
  Emit AI Agent friendly reconstruction contract.

MaterialFamilyUsageStatsExporter
  Batch scan materials.
  Count master/layer/blend usage by active stack and parent family.
  Rank modules by frequency and visual impact hints.
```

### CLI Additions

Keep existing single-material export behavior, but add optional modes:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --game Subnautica2 `
  --paks "..." `
  --mapping "..." `
  --material "/Game/Art/.../MI_CG_RockSmooth_01a" `
  --out "D:\ShaderWP\MI_CG_RockSmooth_01a.bundle" `
  --include-layer-stack `
  --include-master-modules `
  --overwrite `
  --verbose
```

Batch catalog:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --game Subnautica2 `
  --paks "..." `
  --mapping "..." `
  --export-master-material-catalog `
  --out "D:\ShaderWP\Subnautica2_MasterMaterialCatalog"
```

Batch usage stats:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --game Subnautica2 `
  --paks "..." `
  --mapping "..." `
  --scan-material-family-usage `
  --parent "/Game/Materials/_Master/Master/M_LayerStandard" `
  --out "D:\ShaderWP\Subnautica2_M_LayerStandard_Usage"
```

`--semantic-only` and `--context-only` should refresh new layer-aware files if the source cooked JSON is present.

## Unity Material Restore Script Upgrade

Current script:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Tools/unity_material_apply_ue_params.py
```

Current behavior:

```text
Matches by sanitized UE parameter name.
Updates only existing Unity .mat properties by default.
```

Required upgrade:

```text
1. Read analysis/material_layer_parameter_bindings.json.
2. Prefer StableKey -> UnityName mapping over name-only matching.
3. Support name-only fallback for old bundles.
4. Report collisions explicitly.
5. Distinguish GlobalParameter, LayerParameter, BlendParameter.
```

Example:

```powershell
python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_material_apply_ue_params.py `
  --mat "K:\...\MI_CG_RockSmooth_01a.mat" `
  --bundle-dir "D:\ShaderWP\MI_CG_RockSmooth_01a.bundle" `
  --layer-aware `
  --report-out "D:\ShaderWP\MI_CG_RockSmooth_01a.unity_material_restore_report.json"
```

Report additions:

```json
{
  "LayerAware": true,
  "MatchedByStableKey": [],
  "MatchedByLegacyNameFallback": [],
  "Collisions": [],
  "SkippedMissingUnityProperties": []
}
```

## AI Agent Docs Update

Generated bundle docs must tell future Agents:

```text
1. For Material Layer materials, read analysis/unity_layer_reconstruction_contract.json before raw DXIL.
2. Use material_layer_stack.json to determine active layer/blend order.
3. Use material_layer_parameter_bindings.json for Unity Properties and .mat restore mapping.
4. Do not collapse parameters with the same display name if Association/Index differ.
5. Prefer MI-specialized shader reconstruction first; extract reusable modules only after visual parity on representative MIs.
```

Files to update:

```text
README.md
AGENTS.md
WORKFLOW.md
NEXT_TASK.md
PROMPT_NEXT_SESSION.md
agent_context.json
skills/unity6-urp-deferred-shader-reconstruction/SKILL.md
commands/verify_bundle.ps1
```

`--verify-only` should fail if a bundle claims `HasMaterialLayers=true` but these files are missing:

```text
analysis/material_family.json
analysis/material_layer_stack.json
analysis/material_layer_parameter_bindings.json
analysis/unity_layer_reconstruction_contract.json
```

For backward compatibility, this strict check can initially be warning-only until the new pass is stable.

## Recommended Unity Reconstruction Strategy

### Phase 1: MI-specialized Shader

For `MI_CG_RockSmooth_01a`:

```text
Generate a dedicated shader:
  Subnautica2/Reconstructed/M_LayerStandard/MI_CG_RockSmooth_01a

Include:
  core M_LayerStandard path
  MLI_CG_RockSmooth_01a
  MLI_CG_RockSmooth_01b
  ML_LayerTint
  ML_LayerCustomPrim_CurveGradient
  MB_MaskID
  MB_VertexColorOverlay
  MB_CustomPrim_Overlay
```

Advantages:

```text
Higher fidelity for one material.
Smaller AI context than a universal M_LayerStandard.
Clearer property mapping and fewer static branch ambiguities.
```

### Phase 2: Module Extraction

After 3-5 representative MIs:

```text
Extract shared modules:
  M_LayerStandard_Common.hlsl
  Layer_ML_LayerStandard.hlsl
  Layer_ML_LayerTint.hlsl
  Layer_ML_LayerCustomPrim_CurveGradient.hlsl
  Blend_MB_MaskID.hlsl
  Blend_MB_VertexColorOverlay.hlsl
```

### Phase 3: Family Shader

Only after usage stats are stable:

```text
Build M_LayerStandard_Family.shader
Use shader_feature/static defines for common module combinations.
Keep MI-specialized fallback for rare or complex layer stacks.
```

## Implementation Plan

### Stage 0: Schema and Fixtures

Tasks:

```text
1. Add schema examples under docs or test fixtures for:
   material_family.json
   material_layer_stack.json
   material_layer_parameter_bindings.json
   material_static_permutation.json
   unity_layer_reconstruction_contract.json

2. Use MI_CG_RockSmooth_01a.bundle as the first fixture.
3. Add one trimsheet material fixture after confirming its layer stack.
```

Acceptance:

```text
Schemas are reviewed and can express MI_CG_RockSmooth_01a without losing Association/Index.
```

### Stage 1: Layer Stack Parser

Tasks:

```text
1. Parse material.cooked.json for MaterialLayers data.
2. Extract ordered Layers and Blends.
3. Include object name, object path, kind, index, source JSON path.
4. Emit analysis/material_layer_stack.json.
```

Acceptance:

```text
MI_CG_RockSmooth_01a stack matches:
  MLI_CG_RockSmooth_01a
  MLI_CG_RockSmooth_01b
  ML_LayerTint x3
  ML_LayerCustomPrim_CurveGradient
  MB_MaskID
  MB_VertexColorOverlay x3
  MB_CustomPrim_Overlay
```

### Stage 2: Referenced Asset Export

Tasks:

```text
1. Export referenced ML_*/MB_*/MLI_* cooked JSON.
2. Write to source/material_layers, source/material_blends, source/material_layer_instances.
3. Update manifest.json.
4. Add missing asset warnings instead of failing full export.
```

Acceptance:

```text
Bundle contains cooked JSON for all referenced _Master layers/blends and local MLI assets that exist in mounted VFS.
```

### Stage 3: Layer-aware Parameter Bindings

Tasks:

```text
1. Parse ScalarParameterValues, VectorParameterValues, TextureParameterValues, StaticSwitchParameterValues if available.
2. Build StableKey = Association + Index + Name.
3. Merge with UniformExpressionSet values.
4. Preserve source priority:
   MaterialInstanceOverride > StaticPermutation > UniformExpressionSetFinal > ParentDefault
5. Emit analysis/material_layer_parameter_bindings.json.
```

Acceptance:

```text
Parameters with duplicate names but different Association/Index do not collide.
UnityName is deterministic and collision-free.
Skipped/unknown parameters include evidence and reason.
```

### Stage 4: Static Permutation Analyzer

Tasks:

```text
1. Extract direct static parameter values when CUE4Parse exposes them.
2. Infer active features from FunctionInfos and active layer/blend stack.
3. Cross-check with shader inventory and runtime feature classifier.
4. Emit analysis/material_static_permutation.json.
```

Acceptance:

```text
Report includes direct facts and inferred features separately.
No inferred feature is marked as source-level proof.
```

### Stage 5: Master Module Catalog

Tasks:

```text
1. Enumerate /Game/Materials/_Master/Master/M_*.
2. Enumerate /Game/Materials/_Master/Layers/ML_*.
3. Enumerate /Game/Materials/_Master/Blends/MB_*.
4. Export cooked JSON and summary facts.
5. Emit catalog files.
```

Acceptance:

```text
Catalog lists all discoverable master/layer/blend modules from mounted VFS.
Each entry has path, type, parameter names, function dependencies, and export status.
```

### Stage 6: Usage Stats

Tasks:

```text
1. Scan all material instances or all materials under selected parent.
2. Parse parent family and active stack.
3. Count layer/blend/module frequency.
4. Rank modules by usage count and by feature hints.
5. Emit usage stats and reconstruction priority.
```

Acceptance:

```text
Can answer:
  Which ML/MB modules are most common?
  Which parent materials use Material Layers?
  Which representative MIs should be reconstructed first?
```

### Stage 7: Unity Contract and Agent Docs

Tasks:

```text
1. Merge family, stack, bindings, static permutation, semantic maps, deferred contract.
2. Emit unity_layer_reconstruction_contract.json.
3. Update AGENTS.md/WORKFLOW.md/PROMPT_NEXT_SESSION.md generation.
4. Update verify_bundle.ps1 expectations.
```

Acceptance:

```text
Fresh AI Agent can start from contract files and reconstruct a MI-specialized shader without needing chat history.
```

### Stage 8: Unity Material Restore Upgrade

Tasks:

```text
1. Add --layer-aware mode to unity_material_apply_ue_params.py.
2. Read material_layer_parameter_bindings.json.
3. Match StableKey -> UnityName first.
4. Fall back to legacy name matching only when no layer-aware map exists.
5. Report collisions and missing properties.
```

Acceptance:

```text
MI_CG_RockSmooth_01a .mat restore can update both core parameters and layer/blend indexed parameters once Unity shader declares them.
```

## Verification Matrix

### Single Bundle

```text
Input:
  MI_CG_RockSmooth_01a

Expected:
  Export succeeds.
  Verify succeeds.
  material_layer_stack.json contains expected active stack.
  material_layer_parameter_bindings.json contains layer/blend indexed parameters.
  unity_layer_reconstruction_contract.json references required modules.
```

### Master Catalog

```text
Input:
  /Game/Materials/_Master

Expected:
  Catalog includes known modules:
    ML_LayerStandard
    ML_LayerTint
    ML_LayerCustomPrim_CurveGradient
    ML_TrimSheet
    MB_MaskID
    MB_VertexColorOverlay
    MB_CustomPrim_Overlay
    MB_TrimSheet
```

### Restore Script

```text
Input:
  Unity shader with layer-aware Properties.
  MI_CG_RockSmooth_01a .mat.
  MI_CG_RockSmooth_01a.bundle.

Expected:
  Dry run maps by StableKey.
  No duplicate-name collision.
  MissingTextureGuids is empty for imported textures.
  Apply writes .mat backup and updates values.
```

## Risks and Limitations

```text
1. Cooked JSON may not preserve full material graph topology.
2. Static switch values may need inference if CUE4Parse does not expose direct structures.
3. MLI parent layer resolution may be partial.
4. DXIL registers remain anonymous; layer-aware parameter keys improve property mapping, not source-level register naming.
5. UE Material Layers can produce many permutations; universal Unity family shader should be deferred until representative MI coverage is good.
6. UE and Unity Deferred lighting remain non-byte-equivalent; this plan improves material surface/layer logic, not engine light-pass equivalence.
```

## Recommended First Implementation Target

Use `MI_CG_RockSmooth_01a` as the first end-to-end target:

```text
1. Implement material_layer_stack.json.
2. Implement material_layer_parameter_bindings.json.
3. Export referenced ML/MB/MLI cooked JSON.
4. Emit unity_layer_reconstruction_contract.json.
5. Update generated Agent docs.
6. Generate a MI-specialized Unity shader from the new contract.
7. Upgrade material restore script to layer-aware mode.
```

This gives the highest immediate value because it exercises:

```text
M_LayerStandard parent
MLI layer instances
repeated ML_LayerTint
MaskID blend
VertexColorOverlay blend
CustomPrim curve gradient
normal/height/mask/gradient paths
duplicate parameter names with different Association/Index
```

## Done Criteria

The exporter adjustment is complete when:

```text
1. Full export of MI_CG_RockSmooth_01a writes all layer-aware analysis files.
2. A fresh Agent can reconstruct from PROMPT_NEXT_SESSION.md without chat history.
3. Unity Shader Properties preserve UE layer/blend parameters by StableKey.
4. Unity .mat restore can apply values by StableKey.
5. --verify-only validates required files for Material Layer bundles.
6. Master catalog and usage stats can rank modules for broader M_LayerStandard family reconstruction.
```
