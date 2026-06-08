# UE Module-first / MI-driven Unity Reconstruction Plan

本文档定义 Subnautica 2 UE cooked material 到 Unity 6 URP Deferred shader 的推荐长期工作流。

核心策略：

```text
先建立全局 Master / Layer / Blend 模块认知。
每次还原具体 MI 时，只实现它实际用到的模块和分支。
实现后的模块沉淀成可复用 HLSL/ShaderLab 组件。
后续 MI 优先复用已有模块，只补新模块、新分支或新参数路径。
每个新 MI 在写 Unity Shader 前必须先做 shader reuse 评估；参数值不同只生成新 `.mat`，不生成新 shader。
```

这个策略比“每个 MI 都写一份专用 shader”更适合长期工程化；也比“一次性还原完整 M_LayerStandard 万能 shader”更可控。

## Goal

最终目标不是只还原单个材质，而是逐步建立一套可复用的 Unity shader module library：

```text
Assets/Shaders/Subnautica2/MaterialModules/
  Master/
    M_LayerStandard_Common.hlsl
    M_Trimsheet_LayerStandard_Common.hlsl
  Layers/
    ML_LayerStandard.hlsl
    ML_LayerTint.hlsl
    ML_LayerCustomPrim_CurveGradient.hlsl
    ML_TrimSheet.hlsl
  Blends/
    MB_MaskID.hlsl
    MB_VertexColorOverlay.hlsl
    MB_CustomPrim_Overlay.hlsl
    MB_HeightLerp.hlsl
  Runtime/
    SN2MaterialAttributes.hlsl
    SN2LayerParameterAccess.hlsl
    SN2PackedTextureDecode.hlsl
  Registry/
    unity_shader_registry.json
```

每个模块旁边必须有 metadata：

```text
ML_LayerTint.module.json
MB_MaskID.module.json
M_LayerStandard_Common.module.json
```

metadata 记录：

```text
UE source asset path
输入/输出结构
参数 StableKey schema
已实现 static branches
未实现 branches
使用过的 cooked evidence
已验证过的 MI 列表
已知近似/缺口
兼容的 shader reuse key / reuse key hash
```

同时维护 shader registry：

```text
Assets/Shaders/Subnautica2/MaterialModules/Registry/unity_shader_registry.json
```

这个 registry 记录每个已实现 Unity Shader 覆盖的 Master、Layer/Blend stack、static feature set、property schema、已验证 MI。新 MI 还原前必须先查 registry；如果结构兼容，就只创建/回填新的 Unity `.mat`。

## Key Design Decision

### Do Not Start From a Universal Shader

不要一开始就试图生成完整万能版：

```text
M_LayerStandard_Family.shader
```

原因：

```text
1. Material Layers / Blends 分支太多。
2. Static switches 在具体 MI permutation 中才确定。
3. 同名参数会在不同 Layer/Blend Index 下重复。
4. 一次性上下文过大，AI Agent 容易误合并或漏分支。
```

### Do Not Ignore Global Modules

也不要每个 MI 都写独立 shader、完全不沉淀模块。

正确方式是：

```text
Global Catalog -> Module Library -> MI Contract -> Compose MI Shader -> Validate -> Promote/Reuse Module
```

## Terminology

```text
Master
  /Game/Materials/_Master/Master/M_*
  材质家族入口，例如 M_LayerStandard、M_Trimsheet_LayerStandard。

Layer
  /Game/Materials/_Master/Layers/ML_*
  可复用 Material Layer graph fragment。

Blend
  /Game/Materials/_Master/Blends/MB_*
  可复用 Material Layer Blend graph fragment。

Layer Instance
  MLI_*
  具体 Layer 参数实例。可能不在 _Master/Layers 目录下。

MI
  MaterialInstanceConstant。
  决定实际 Parent、active layer stack、blend stack、static permutation、参数覆盖。
```

## Required Exporter Outputs

Exporter 需要支持两类输出。

### Project-level Module Catalog

输出目录示例：

```text
D:\ShaderWP\Subnautica2_ModuleCatalog\
```

需要生成：

```text
analysis/master_material_catalog.json
analysis/material_layer_module_catalog.json
analysis/material_blend_module_catalog.json
analysis/material_layer_instance_catalog.json
analysis/master_material_usage_stats.json
analysis/material_layer_usage_stats.json
analysis/material_blend_usage_stats.json
analysis/material_family_reconstruction_priority.json
source/master_materials/*.cooked.json
source/material_layers/*.cooked.json
source/material_blends/*.cooked.json
```

用途：

```text
1. 让 Agent 知道全局有哪些 Master / Layer / Blend。
2. 提前识别模块参数、函数依赖和常见组合。
3. 按使用频率决定优先还原哪些模块。
```

### MI Bundle Layer-aware Contract

每个 MI bundle 需要生成：

```text
analysis/material_family.json
analysis/material_layer_stack.json
analysis/material_layer_parameter_bindings.json
analysis/material_static_permutation.json
analysis/material_layer_function_dependencies.json
analysis/unity_shader_reuse_key.json
analysis/unity_shader_reuse_candidates.json
analysis/unity_shader_assignment.json
analysis/unity_layer_reconstruction_contract.json
source/material_layers/*.cooked.json
source/material_layer_instances/*.cooked.json
source/material_blends/*.cooked.json
```

用途：

```text
1. 告诉 Agent 当前 MI 实际使用哪些模块。
2. 告诉 Agent 每个模块实例的 Layer/Blend Index。
3. 告诉 Agent 每个 UE 参数应该映射到哪个 Unity property。
4. 告诉 Agent 哪些 static branches 在当前 MI 中已启用。
5. 告诉 Agent 当前 MI 是否可以复用已有 Unity Shader。
```

## Stable Parameter Key

UE Material Layers 参数不能只按名字处理。必须使用：

```text
ParameterName + Association + Index
```

标准 key：

```text
GlobalParameter:-1:SelectionColor
LayerParameter:0:Normal Intensity
LayerParameter:5:Gradient Curve
BlendParameter:0:Threshold
BlendParameter:3:Vertex Paint - Opacity
```

Unity property 内部名：

```text
_Global_SelectionColor
_Layer0_Normal_Intensity
_Layer5_Gradient_Curve
_Blend0_Threshold
_Blend3_Vertex_Paint_Opacity
```

规则：

```text
1. Unity display name 可以保留 UE 原始名字。
2. Unity internal name 必须 collision-free。
3. 同名 UE 参数只要 Association/Index 不同，就不能合并。
4. .mat 回填必须优先使用 StableKey 映射。
```

## Unity Runtime Struct

所有 Layer / Blend 模块都应该读写统一的 Unity-side material attributes struct。

示例：

```hlsl
struct SN2MaterialAttributes
{
    float3 baseColor;
    float metallic;
    float roughness;
    float specular;
    float3 normalTS;
    float opacity;
    float occlusion;
    float3 emission;
    float subsurface;
    float pixelDepthOffset;
    float3 worldPositionOffset;
    float4 customData0;
    float4 customData1;
};
```

模块接口示例：

```hlsl
SN2MaterialAttributes ML_LayerTint_Apply(
    SN2MaterialAttributes input,
    SN2LayerContext ctx,
    SN2LayerTintParams p);

SN2MaterialAttributes MB_MaskID_Blend(
    SN2MaterialAttributes baseLayer,
    SN2MaterialAttributes topLayer,
    SN2BlendContext ctx,
    SN2MaskIDParams p);
```

不要让模块直接依赖 Unity URP `SurfaceData`。统一由 Master composer 在最后做：

```text
SN2MaterialAttributes -> URP SurfaceData/InputData/GBuffer
```

## Module Metadata

每个模块需要有 `.module.json`。

示例：

```json
{
  "Schema": "sn2-unity-material-module/v1",
  "ModuleName": "ML_LayerTint",
  "ModuleKind": "Layer",
  "UESourcePath": "/Game/Materials/_Master/Layers/ML_LayerTint",
  "UnitySource": "Assets/Shaders/Subnautica2/MaterialModules/Layers/ML_LayerTint.hlsl",
  "Inputs": ["SN2MaterialAttributes", "SN2LayerContext", "SN2LayerTintParams"],
  "Outputs": ["SN2MaterialAttributes"],
  "Parameters": [
    {
      "UEName": "Base Colour Tint",
      "Type": "Vector",
      "StableKeyPattern": "LayerParameter:{Index}:Base Colour Tint"
    }
  ],
  "ImplementedBranches": [],
  "UnimplementedBranches": [],
  "Evidence": [
    "source/material_layers/ML_LayerTint.cooked.json",
    "analysis/material_layer_function_dependencies.json"
  ],
  "ValidatedMaterials": [
    "MI_CG_RockSmooth_01a"
  ],
  "KnownLimitations": []
}
```

## Shader Reuse Model

多个 MI 如果只是参数值、贴图、颜色、roughness 等数据不同，应该共用同一个 Unity Shader，生成不同 Unity `.mat`。

不能用“MI 名字不同”作为新建 shader 的理由。必须比较结构化 reuse key。

### Reuse Key

`analysis/unity_shader_reuse_key.json`：

```json
{
  "Schema": "unity-shader-reuse-key/v1",
  "Material": "/Game/Art/.../MI_CG_RockSmooth_01a",
  "Parent": "/Game/Materials/_Master/Master/M_LayerStandard",
  "BlendMode": "BLEND_Masked",
  "RenderingPath": "Unity6URPDeferred",
  "TwoSided": false,
  "UsesWPO": true,
  "UsesPDO": false,
  "LayerModules": [
    "ML_LayerStandard",
    "ML_LayerStandard",
    "ML_LayerTint",
    "ML_LayerTint",
    "ML_LayerTint",
    "ML_LayerCustomPrim_CurveGradient"
  ],
  "BlendModules": [
    "MB_MaskID",
    "MB_VertexColorOverlay",
    "MB_VertexColorOverlay",
    "MB_VertexColorOverlay",
    "MB_CustomPrim_Overlay"
  ],
  "StaticFeatureSet": [
    "UseNormalFromHeightmap",
    "UseVertexColorOverlay",
    "UseCustomPrimCurveGradient"
  ],
  "RequiredModuleBranches": [
    "ML_LayerStandard:packed_BCM_NRH",
    "MB_MaskID:color_id_mask",
    "MB_VertexColorOverlay:overlay"
  ],
  "PropertySchemaHash": "sha256-of-stable-property-schema",
  "ModuleBranchHash": "sha256-of-module-and-branch-set",
  "ShaderReuseClass": "M_LayerStandard.masked.layerstack.hash"
}
```

Reuse key 不能包含普通参数值；只有会改变 shader 结构或编译分支的 static value 才能进入 key。

### Reuse Candidates

`analysis/unity_shader_reuse_candidates.json`：

```json
{
  "Schema": "unity-shader-reuse-candidates/v1",
  "Material": "MI_CG_RockSmooth_01a",
  "Candidates": [
    {
      "UnityShader": "Subnautica2/Reconstructed/M_LayerStandard/LayerStack_7A1F",
      "MatchKind": "exact",
      "Compatibility": 1.0,
      "Reason": "Parent, render state, layer modules, blend modules, static features, and property schema match."
    },
    {
      "UnityShader": "Subnautica2/Reconstructed/M_LayerStandard/LayerStack_Base",
      "MatchKind": "extendable",
      "Compatibility": 0.74,
      "MissingBranches": [
        "ML_LayerCustomPrim_CurveGradient"
      ],
      "Reason": "Base modules match but one layer branch is not implemented."
    }
  ]
}
```

### Shader Assignment

`analysis/unity_shader_assignment.json`：

```json
{
  "Schema": "unity-shader-assignment/v1",
  "Material": "MI_CG_RockSmooth_01a",
  "Decision": "reuse_existing|extend_existing|create_new",
  "AssignedUnityShader": "Subnautica2/Reconstructed/M_LayerStandard/LayerStack_7A1F",
  "Reason": "Exact reuse key match.",
  "RequiredActions": [
    "Create Unity .mat.",
    "Run layer-aware material restore."
  ]
}
```

### Shader Registry

`Assets/Shaders/Subnautica2/MaterialModules/Registry/unity_shader_registry.json`：

```json
{
  "Schema": "sn2-unity-shader-registry/v1",
  "Shaders": [
    {
      "Id": "LayerStack_7A1F",
      "UnityShaderName": "Subnautica2/Reconstructed/M_LayerStandard/LayerStack_7A1F",
      "ShaderFile": "Assets/Shaders/Subnautica2/Reconstructed/M_LayerStandard/LayerStack_7A1F.shader",
      "Parent": "/Game/Materials/_Master/Master/M_LayerStandard",
      "ReuseKeyHash": "7A1F...",
      "SupportedLayerModules": [],
      "SupportedBlendModules": [],
      "SupportedStaticFeatures": [],
      "PropertySchemaHash": "...",
      "ValidatedMaterials": [
        "MI_CG_RockSmooth_01a"
      ],
      "KnownLimitations": []
    }
  ]
}
```

## Workflow Overview

### Step 1: Export Global Module Catalog

Run exporter in catalog mode:

```powershell
cd D:\Github\FModel

dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --game Subnautica2 `
  --paks "D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\Subnautica2\Content\Paks" `
  --mapping "D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\5.6.1-114707+++Project+SN2-Release-Hotfix-Live-Subnautica2.usmap" `
  --export-master-material-catalog `
  --out "D:\ShaderWP\Subnautica2_ModuleCatalog" `
  --verbose
```

Expected output:

```text
D:\ShaderWP\Subnautica2_ModuleCatalog\analysis\master_material_catalog.json
D:\ShaderWP\Subnautica2_ModuleCatalog\analysis\material_layer_module_catalog.json
D:\ShaderWP\Subnautica2_ModuleCatalog\analysis\material_blend_module_catalog.json
```

### Step 2: Export Usage Stats

Run parent-family usage scan:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --game Subnautica2 `
  --paks "D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\Subnautica2\Content\Paks" `
  --mapping "D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\5.6.1-114707+++Project+SN2-Release-Hotfix-Live-Subnautica2.usmap" `
  --scan-material-family-usage `
  --parent "/Game/Materials/_Master/Master/M_LayerStandard" `
  --out "D:\ShaderWP\Subnautica2_M_LayerStandard_Usage" `
  --verbose
```

Use this to choose representative MIs:

```text
1. Common environment rock/coral material.
2. Trimsheet material.
3. Hard-surface material.
4. Character/creature material.
5. VFX/translucent material only after opaque path is stable.
```

### Step 3: Export First MI Bundle

Example:

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

Verify:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --verify-only "K:\WorkSpace\ShaderReverse\MI_CG_RockSmooth_01a\MI_CG_RockSmooth_01a.bundle" `
  --verbose
```

### Step 4: Start Unity Reconstruction Session

```powershell
cd K:\WorkSpace\ShaderReverse\MI_CG_RockSmooth_01a\MI_CG_RockSmooth_01a.bundle
```

Then:

```text
/Goal PROMPT_NEXT_SESSION.md
```

Required read order:

```text
AGENTS.md
WORKFLOW.md
analysis/ai_context_pack.md
analysis/unity_layer_reconstruction_contract.json
analysis/material_layer_stack.json
analysis/material_layer_parameter_bindings.json
analysis/material_static_permutation.json
analysis/unity_shader_reuse_key.json
analysis/unity_shader_reuse_candidates.json
analysis/unity_shader_assignment.json
analysis/material_family.json
analysis/unity_deferred_reconstruction_contract.json
parameters/material_parameters.json
parameters/textures.json
source/material_layers/*.cooked.json for used modules only
source/material_blends/*.cooked.json for used modules only
source/material_layer_instances/*.cooked.json for used modules only
selected shaders/*.dxil.ll only when evidence is needed
```

Do not load:

```text
all shaders/*.dxil.ll
all module catalog cooked JSON
all raw RenderDoc buffers
```

### Step 5: Build or Reuse Modules

Agent checks module library first:

```text
Assets/Shaders/Subnautica2/MaterialModules/**/*.module.json
```

For each required module in `unity_layer_reconstruction_contract.json`:

```text
If module exists and supports required branch:
  reuse it.

If module exists but branch/parameter path is missing:
  extend module and update .module.json.

If module does not exist:
  implement module from cooked JSON/DXIL evidence and add .module.json.
```

Before writing or modifying Unity shader code, Agent must also check shader reuse:

```text
1. Read analysis/unity_shader_reuse_key.json.
2. Read analysis/unity_shader_reuse_candidates.json.
3. Read analysis/unity_shader_assignment.json.
4. Read Assets/Shaders/Subnautica2/MaterialModules/Registry/unity_shader_registry.json if it exists.
5. If assignment is reuse_existing:
   do not create a new shader; create/assign .mat and restore parameters.
6. If assignment is extend_existing:
   extend only the missing modules/branches and regression-check already validated MIs.
7. If assignment is create_new:
   create a new module-based shader and register it.
```

For `MI_CG_RockSmooth_01a`, likely first modules:

```text
Master:
  M_LayerStandard_Common

Layers:
  ML_LayerStandard
  MLI_CG_RockSmooth_01a adapter
  MLI_CG_RockSmooth_01b adapter
  ML_LayerTint
  ML_LayerCustomPrim_CurveGradient

Blends:
  MB_MaskID
  MB_VertexColorOverlay
  MB_CustomPrim_Overlay
```

### Step 6: Compose MI Shader

Generated shader should be module-based. It can be MI-specific or shared by multiple MIs depending on `analysis/unity_shader_assignment.json`:

```text
Assets/Shaders/Subnautica2/Reconstructed/M_LayerStandard/MI_CG_RockSmooth_01a.shader
Assets/Shaders/Subnautica2/Reconstructed/M_LayerStandard/MI_CG_RockSmooth_01a_Input.hlsl
```

If reuse key indicates the shader can be shared, prefer a structural shader name:

```text
Assets/Shaders/Subnautica2/Reconstructed/M_LayerStandard/LayerStack_7A1F.shader
Assets/Shaders/Subnautica2/Reconstructed/M_LayerStandard/LayerStack_7A1F_Input.hlsl
```

Then assign multiple Unity `.mat` files to the same shader.

The shader should:

```text
1. Target Unity 6 URP Deferred.
2. Include DOTS instancing support.
3. Use UniversalGBuffer pass.
4. Reuse Unity URP Deferred lighting.
5. Compose active Layer/Blend stack using reusable modules.
6. Generate Properties from material_layer_parameter_bindings.json.
7. Use StableKey-derived property names.
8. Document unsupported UE runtime features.
```

Composition pseudo-flow:

```hlsl
SN2MaterialAttributes a0 = ML_LayerStandard_Evaluate(ctx, layer0Params);
SN2MaterialAttributes a1 = ML_LayerStandard_Evaluate(ctx, layer1Params);
SN2MaterialAttributes tint2 = ML_LayerTint_Evaluate(ctx, layer2Params);

SN2MaterialAttributes out01 = MB_MaskID_Blend(a0, a1, ctx, blend0Params);
SN2MaterialAttributes out012 = MB_VertexColorOverlay_Blend(out01, tint2, ctx, blend1Params);
SN2MaterialAttributes finalAttrs = MB_CustomPrim_Overlay_Blend(out012, curveGradient, ctx, blend4Params);

ConvertSN2AttributesToURPSurfaceData(finalAttrs, surfaceData);
```

### Step 7: Create Unity Material

Create `.mat` with same asset name:

```text
MI_CG_RockSmooth_01a.mat
```

Assign generated shader:

```text
Subnautica2/Reconstructed/M_LayerStandard/LayerStack_7A1F
```

If `unity_shader_assignment.json.Decision` is `reuse_existing`, this is the first Unity operation for that MI; shader code should not be changed.

### Step 8: Restore Material Values

Dry run:

```powershell
python D:\Github\FModel\CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_material_apply_ue_params.py `
  --mat "K:\...\MI_CG_RockSmooth_01a.mat" `
  --bundle-dir "K:\WorkSpace\ShaderReverse\MI_CG_RockSmooth_01a\MI_CG_RockSmooth_01a.bundle" `
  --layer-aware `
  --report-out "K:\WorkSpace\ShaderReverse\MI_CG_RockSmooth_01a\unity_material_restore_report.json"
```

Review report:

```text
MatchedByStableKey must cover core + layer + blend parameters.
Collisions must be empty.
MissingTextureGuids must be empty or documented.
SkippedMissingUnityProperties must be empty or explicitly accepted.
```

Apply:

```powershell
python D:\Github\FModel\CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_material_apply_ue_params.py `
  --mat "K:\...\MI_CG_RockSmooth_01a.mat" `
  --bundle-dir "K:\WorkSpace\ShaderReverse\MI_CG_RockSmooth_01a\MI_CG_RockSmooth_01a.bundle" `
  --layer-aware `
  --report-out "K:\WorkSpace\ShaderReverse\MI_CG_RockSmooth_01a\unity_material_restore_report.json" `
  --apply
```

### Step 9: Validate and Promote Modules

Validation checklist:

```text
1. Shader compiles in Unity 6.
2. Material property count matches material_layer_parameter_bindings.json.
3. Textures bind correctly.
4. GBuffer pass runs in URP Deferred.
5. Visual result is compared against screenshot/RenderDoc evidence when available.
6. Known approximations are documented.
7. Module metadata records MI_CG_RockSmooth_01a as a validated material.
```

Promotion rule:

```text
If a module was implemented only for one MI but is source-backed by _Master/Layer or _Master/Blend,
store it under MaterialModules and mark current coverage in .module.json.

Do not hide missing branches. Record them in UnimplementedBranches.
```

### Step 10: Repeat for Next MI

For the next representative MI:

```text
1. Export layer-aware bundle.
2. Read its contract.
3. Read shader reuse key/candidates/assignment.
4. If an existing shader is compatible, reuse it and only create/restore the .mat.
5. Reuse existing modules.
6. Extend modules only for new branches.
7. Compose a new shader only when assignment says create_new or extend_existing requires a new variant.
8. Restore .mat values.
9. Validate.
10. Update module metadata and shader registry.
```

After several MIs, generate:

```text
M_LayerStandard_Family.shader
```

only when module coverage is high enough.

## Exporter Implementation Plan

### Phase A: Catalog Support

Implement:

```text
MasterMaterialCatalogExporter
MaterialLayerModuleCatalogExporter
MaterialBlendModuleCatalogExporter
MaterialFamilyUsageStatsExporter
UnityShaderRegistryReader
```

Acceptance:

```text
Can list all discoverable _Master/Master, _Master/Layers, _Master/Blends assets.
Can rank modules by usage across material instances.
```

### Phase B: MI Layer Contract

Implement:

```text
MaterialFamilyAnalyzer
MaterialLayerStackAnalyzer
MaterialLayerAssetExporter
LayerAwareParameterBindingAnalyzer
MaterialStaticPermutationAnalyzer
UnityShaderReuseKeyWriter
UnityShaderReuseCandidateAnalyzer
UnityShaderAssignmentWriter
UnityLayerContractWriter
```

Acceptance:

```text
MI_CG_RockSmooth_01a.bundle contains:
  analysis/material_family.json
  analysis/material_layer_stack.json
  analysis/material_layer_parameter_bindings.json
  analysis/material_static_permutation.json
  analysis/unity_shader_reuse_key.json
  analysis/unity_shader_reuse_candidates.json
  analysis/unity_shader_assignment.json
  analysis/unity_layer_reconstruction_contract.json
```

### Phase C: Agent Workspace Update

Update generated:

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

New rules:

```text
For Material Layer materials, read unity_layer_reconstruction_contract.json before raw DXIL.
Do not merge same-name parameters across different Association/Index.
Use module-first / MI-driven validation.
Reuse module library before writing new module code.
Evaluate shader reuse before creating a new Unity Shader.
Parameter differences create different .mat files, not different shaders.
```

### Phase D: Unity Restore Script

Upgrade:

```text
unity_material_apply_ue_params.py
```

Add:

```text
--layer-aware
--module-metadata-root
StableKey matching
collision report
legacy fallback report
```

Acceptance:

```text
Can restore MI_CG_RockSmooth_01a values into a layer-aware Unity .mat without name collisions.
```

## AI Agent Operating Rules

Future reconstruction Agents must follow:

```text
1. Start from MI contract, not from all DXIL files.
2. Check existing MaterialModules before implementing a module.
3. Check unity_shader_reuse_key/candidates/assignment before writing shader code.
4. Reuse an existing Unity Shader when the structural reuse key matches.
5. Implement modules as reusable files with metadata.
6. Compose shader from modules only when reuse is impossible or extension is required.
7. Use StableKey for properties and .mat restore.
8. Record all assumptions and unimplemented branches.
9. Do not port UE Deferred LightPass into material shader unless lightpass audit or RenderDoc proves custom game lighting.
10. Prefer Unity URP Deferred lighting for first-pass material reconstruction.
```

## First Target

Use `MI_CG_RockSmooth_01a` for the first end-to-end validation because it exercises:

```text
M_LayerStandard parent
MLI layer instances
repeated ML_LayerTint
ML_LayerCustomPrim_CurveGradient
MB_MaskID
MB_VertexColorOverlay
MB_CustomPrim_Overlay
Normal/height/mask/gradient logic
duplicate parameter names across Association/Index
```

Expected first deliverables:

```text
MaterialModules/Master/M_LayerStandard_Common.hlsl
MaterialModules/Layers/ML_LayerStandard.hlsl
MaterialModules/Layers/ML_LayerTint.hlsl
MaterialModules/Layers/ML_LayerCustomPrim_CurveGradient.hlsl
MaterialModules/Blends/MB_MaskID.hlsl
MaterialModules/Blends/MB_VertexColorOverlay.hlsl
MaterialModules/Blends/MB_CustomPrim_Overlay.hlsl
Reconstructed/M_LayerStandard/MI_CG_RockSmooth_01a.shader
Reconstructed/M_LayerStandard/MI_CG_RockSmooth_01a_Input.hlsl
```

## Done Criteria

This plan is successful when:

```text
1. Exporter can produce project-level module catalog and MI-level layer-aware bundle.
2. Fresh Agent can reconstruct MI_CG_RockSmooth_01a from generated docs without chat history.
3. Unity shader uses reusable modules, not duplicated one-off code.
4. Unity Properties map to UE StableKey values.
5. .mat restore uses StableKey and has no collisions.
6. Implemented modules are reusable by the next MI.
7. Missing static branches and unsupported runtime features are explicit in module metadata.
8. A new MI with the same structural layer/blend/static feature set reuses the existing Unity Shader and only gets a new .mat.
9. Shader registry records which MIs validated each shared shader.
```
