# Goal: Module-first MI-driven Unity Shader Reconstruction

Objective: reconstruct a UE cooked Material Instance bundle into Unity 6 URP Deferred using reusable Master/Layer/Blend modules, StableKey properties, and shader reuse assignment.

Use this goal after `Doc/UE_Cooked_Material_Bundle_Export_Goal.md` has produced a verified bundle.

## Invocation

Preferred forms:

```text
/Goal D:\Github\FModel\Doc\UE_Module_First_MI_Driven_Unity_Reconstruction_Goal.md

/Goal D:\Github\FModel\Doc\UE_Module_First_MI_Driven_Unity_Reconstruction_Goal.md Bundle=K:\WorkSpace\ShaderReverse\MI_Name\MI_Name.bundle UnityRoot=K:\WorkSpace\trunk\ExportedProject

/Goal D:\Github\FModel\Doc\UE_Module_First_MI_Driven_Unity_Reconstruction_Goal.md Mat=K:\WorkSpace\trunk\ExportedProject\Assets\...\MI_Name.mat
```

If invoked from inside a bundle directory, use the current directory as `Bundle`.

If invoked only from a bundle prompt, for example:

```text
/Goal MI_CG_RockSmooth_01a.bundle\PROMPT_NEXT_SESSION.md
```

do not infer a full Unity project scan and do not process Unity `.mat` files. In this mode the Agent should read the bundle and only touch already-known target shader output paths. Unity project asset access is opt-in and scoped by explicit `UnityRoot=...`. Unity `.mat` creation, shader assignment, inspection, and property restore are reserved for `Doc\UE_Unity_Material_Property_Restore_Goal.md`.

## Required Inputs

Bundle must contain:

```text
AGENTS.md
WORKFLOW.md
analysis/ai_context_pack.md
analysis/reconstruction_entrypoints.json
analysis/unity_layer_reconstruction_contract.json
analysis/material_layer_stack.json
analysis/material_layer_parameter_bindings.json
analysis/material_static_permutation.json
analysis/unity_shader_reuse_key.json
analysis/unity_shader_reuse_candidates.json
analysis/unity_shader_assignment.json
analysis/unity_deferred_reconstruction_contract.json
parameters/material_parameters.json
parameters/textures.json
```

Optional project catalog:

```text
analysis/master_material_catalog.json
analysis/material_layer_module_catalog.json
analysis/material_blend_module_catalog.json
analysis/material_family_reconstruction_priority.json
```

## Read Order

Read in this order:

```text
AGENTS.md
WORKFLOW.md
analysis/ai_context_pack.md
analysis/reconstruction_entrypoints.json
analysis/unity_layer_reconstruction_contract.json
analysis/material_layer_stack.json
analysis/material_layer_parameter_bindings.json
analysis/material_static_permutation.json
analysis/unity_shader_reuse_key.json
analysis/unity_shader_reuse_candidates.json
analysis/unity_shader_assignment.json
analysis/unity_deferred_reconstruction_contract.json
analysis/semantic_binding_map.json
parameters/material_parameters.json
parameters/textures.json
```

Do not start by loading every `shaders/*.dxil.ll`.

## Decision Rules

Before writing shader code:

```text
1. Read analysis/unity_shader_assignment.json.
2. If Decision=reuse_existing:
   do not create a new Unity Shader.
   record the assigned existing shared shader.
   stop shader generation.
   do not create, assign, inspect, or restore Unity .mat files in this goal.
3. If Decision=extend_existing:
   extend only missing modules/branches.
   regression-check already validated MIs for that shader.
   if no missing module branches are required, reuse the assigned shader and register the bundle as validated.
4. If Decision=create_new:
   create a structural shader from reusable modules.
   register it after compile/validation.
```

Parameter values do not create new shaders. They create different Unity `.mat` files.

## Fidelity Bar

This goal is not complete if it only produces a compileable scaffold.

Do not stop at:

```text
Properties block only
BCM/NRH texture sampling only
tint only
rough normal decode only
rough mask lerp only
URP Deferred/DOTS boilerplate only
```

The Agent must reconstruct shader/module logic from bundle evidence. For every evidenced Master/Layer/Blend module and static feature branch referenced by the MI, do one of:

```text
Implement it in the Unity module/shader.
Map it to an existing implemented module branch and cite that branch.
Mark it incomplete with exact missing evidence and expected visual impact.
```

Required evidence sources:

```text
analysis/unity_layer_reconstruction_contract.json
analysis/material_layer_stack.json
analysis/material_layer_parameter_bindings.json
analysis/material_static_permutation.json
analysis/reconstruction_entrypoints.json
analysis/ai_context_pack.md/json
analysis/semantic_binding_map.json
analysis/texture_register_statistics.json
source/material_functions/*.cooked.json, when present
analysis/module_formula_evidence.json, when present
analysis/dxil_formula_evidence.json, when present
analysis/curve_atlas_metadata.json, when present
selected shaders/*.dxil.ll, only for chosen entrypoints and specific formula/binding questions
analysis/renderdoc/renderdoc_runtime_overlay.md/json, if present
```

Visible logic that must be implemented or explicitly marked incomplete when present in the bundle:

```text
ML_LayerStandard / layer standard sampling and remap
MLI_* layer instance overrides
ML_LayerTint
ML_LayerCustomPrim_CurveGradient
MB_MaskID
MB_VertexColorOverlay
MB_CustomPrim_Overlay
height lerp / height contrast / height power
normal intensity / normal blend / normal from height
roughness and metallic remap
mask channel selection
vertex color influence
curve/gradient atlas influence
UV tiling / UV offset / object scale / triplanar-like branches, when evidenced
```

A shader that compiles and preserves all properties but omits evidenced layer/blend/function behavior must be reported as incomplete, not complete.

## Unity Project Access Boundary

The reconstruction Agent must not recursively inspect the Unity project `Assets` tree by default.

Default bundle-prompt mode:

```text
Allowed:
  Bundle directory
  analysis/*
  parameters/*
  selected shaders/*.dxil.ll
  already-known generated shader output path, if the assignment or user provides one

Forbidden:
  recursive scan of K:\WorkSpace\trunk\ExportedProject\Assets
  recursive scan of any Unity Assets tree
  analysis of unrelated existing Unity .mat files
  analysis of unrelated existing Unity shaders
```

Shader reuse must be decided from:

```text
analysis/unity_shader_assignment.json
analysis/unity_shader_reuse_candidates.json
analysis/unity_shader_reuse_key.json
```

If this evidence is missing, stale, or inconclusive, report that `UnityRoot` or a refreshed shader registry is required. Do not scan Unity `Assets` to discover candidates.

When `UnityRoot=...` is explicitly provided, restrict Unity project access to the target shader implementation surface:

```text
Assets/Shaders/Subnautica2/MaterialModules
Assets/Shaders/Subnautica2/Reconstructed
Assets/Shaders/Subnautica2/MaterialModules/Registry/unity_shader_registry.json
assigned .shader.meta
Assets/Editor/SN2ShaderValidation.cs
```

Do not inspect unrelated Unity project assets.

Only read Unity `.mat` files or texture asset folders in the separate material restore goal:

```text
Doc\UE_Unity_Material_Property_Restore_Goal.md
```

`Mat=...`, `AssetsRoot=...`, and `TextureSearchRoot=...` are material-restore arguments, not shader-reconstruction work for `PROMPT_NEXT_SESSION.md`.

## Module Rules

Use this target library:

```text
Assets/Shaders/Subnautica2/MaterialModules/
  Master/
  Layers/
  Blends/
  Runtime/
  Registry/unity_shader_registry.json
```

For every module in `analysis/unity_layer_reconstruction_contract.json`:

```text
If module exists and supports the required branch:
  reuse it.

If module exists but branch is missing:
  extend the module and update its .module.json.

If module does not exist:
  implement the module and write a .module.json.
```

Do not hide missing branches. Record them in `.module.json`.

## Properties Contract

Generate Unity Shader `Properties` from:

```text
analysis/material_layer_parameter_bindings.json
```

Rules:

```text
1. Use StableKey-derived Unity internal names.
2. Preserve UE display names, parameter types, defaults, texture references, and override metadata.
3. Do not merge same-name parameters across Association/Index.
4. Use DOTS instancing-compatible property declarations.
```

Example names:

```text
_Layer5_Gradient_Curve
_Blend0_Color_Mask_Input
_Global_SelectionColor
```

## Deferred Lighting

Target:

```text
Unity 6 URP Deferred
DOTS instancing enabled
```

Reuse Unity URP Deferred lighting unless `analysis/lightpass_audit.json` or RenderDoc evidence proves custom game lighting. Do not port UE deferred LightPass code into the material shader by default.

## Semantic Debug Outputs

When writing or extending a Unity shader for this workflow, add semantic debug support so the later visual validation goal can compare material channels instead of relying only on final lit screenshots.

Use the template:

```text
CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_shader_validation\SN2SemanticDebug.hlsl.txt
```

Rules:

```text
1. Add `_SN2DebugMode` to the shader `Properties` block.
2. Add `int _SN2DebugMode;` to the shader's existing UnityPerMaterial CBUFFER.
3. Fill `SN2DebugSurface` from the same material/layer evaluation code used by the UniversalGBuffer pass.
4. Support at least the bundle-visible channels: BaseColor, NormalWS or NormalTS, Roughness/Smoothness, Metallic, AmbientOcclusion, Alpha/OpacityMask, LayerBlend, HeightBlend when the shader computes them.
5. If a channel is not implemented, document the missing evidence and visual impact in the reconstruction report.
```

## Material Restore

After the shader and `.mat` exist, run:

```powershell
cd D:\Github\FModel

python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_material_apply_ue_params.py `
  --mat "UNITY_MAT" `
  --bundle-dir "BUNDLE" `
  --layer-aware `
  --assign-shader-meta "ASSIGNED_SHADER.shader.meta" `
  --report-out "RESTORE_REPORT"
```

Apply only after the dry-run report is correct:

```powershell
python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_material_apply_ue_params.py `
  --mat "UNITY_MAT" `
  --bundle-dir "BUNDLE" `
  --layer-aware `
  --assign-shader-meta "ASSIGNED_SHADER.shader.meta" `
  --report-out "RESTORE_REPORT" `
  --apply `
  --apply-confirm WRITE_MAT
```

If the Unity `.mat` does not exist yet:

```powershell
python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_material_apply_ue_params.py `
  --mat "UNITY_MAT" `
  --bundle-dir "BUNDLE" `
  --layer-aware `
  --assign-shader-meta "ASSIGNED_SHADER.shader.meta" `
  --add-missing `
  --create-if-missing `
  --report-out "RESTORE_REPORT" `
  --apply `
  --apply-confirm WRITE_MAT
```

Expected report:

```text
Collisions is empty.
MatchedByStableKey covers expected core/layer/blend properties.
MatchedByLegacyNameFallback is zero or explicitly explained.
MissingTextureGuids is empty or accepted.
SkippedMissingUnityProperties is empty or explained.
```

After material restore succeeds with `Apply`, run semantic visual validation when the workspace has original-game references under `VisualRefs` or the user requests visual validation:

```text
/goal UE_Unity_Material_Semantic_Visual_Validation_Goal.md
```

Use the bundle-local goal when present; use `D:\Github\FModel\Doc\UE_Unity_Material_Semantic_Visual_Validation_Goal.md` as fallback. Read `visual_validation_report.md/json` and `visual_validation_advice.json` before raw screenshot or diff inspection. Prefer decoded semantic GBuffer/material channel captures over raw GBuffer attachments. A final lit screenshot alone is not enough to declare high-fidelity reconstruction.

## Registry Validation

When a second MI reuses or extends an existing shader without writing new shader code, register the validated bundle:

```powershell
python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\generate_unity_layer_stack_shader.py `
  --bundle-dir "BUNDLE" `
  --unity-root "UNITY_PROJECT_ROOT" `
  --register-validated-bundle
```

This updates:

```text
Assets/Shaders/Subnautica2/MaterialModules/Registry/unity_shader_registry.json
```

Expected registry evidence:

```text
ValidatedMaterials includes the MI path.
CompatibleReuseKeyHashes records compatible non-exact reuse keys.
ObservedPropertySchemaHashes records property schema variants.
LastValidation records Decision, Reason, Bundle, and Material.
```

## Unity Shader Validation

Use Unity 6 for shader import/compile validation.

The validation script template lives at:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Tools/unity_shader_validation/SN2ShaderValidation.cs.txt
```

Copy it into the Unity project as:

```text
Assets/Editor/SN2ShaderValidation.cs
```

Batchmode form, when the project is not already open:

```powershell
$env:SN2_SHADER_VALIDATION_SHADER="Assets/Shaders/Subnautica2/Reconstructed/M_LayerStandard/LayerStack_C43830F3.shader"
$env:SN2_SHADER_VALIDATION_REPORT="D:\Tmp\ShaderBundles\MI_Name.bundle\unity_shader_validation_report.json"
& "K:\Unity\6000.0.62f1\Editor\Unity.exe" `
  -batchmode `
  -quit `
  -projectPath "K:\WorkSpace\trunk\ExportedProject" `
  -executeMethod SN2ShaderValidation.Run `
  -logFile "D:\Tmp\ShaderBundles\MI_Name.bundle\unity_shader_validation_editor.log"
```

If the project is already open, run this menu item in the open Editor:

```text
Tools/Subnautica2/Validate LayerStack Shader
```

## Completion Criteria

The task is complete when:

```text
1. Shader reuse assignment was honored.
2. Required Master/Layer/Blend modules exist or were extended with implemented logic, not only metadata.
3. Unity shader targets Unity 6 URP Deferred and supports DOTS instancing.
4. Unity Properties are StableKey-derived and one-to-one with UE parameters.
5. Every evidenced module/static branch is implemented or listed as incomplete with missing evidence and visual impact.
6. Selected DXIL/dataflow evidence was used for concrete formula or binding questions instead of relying only on parameter names.
7. Runtime-only UE features are documented as direct, approximate, or omitted.
8. Shader registry/module metadata were updated when a shader/module was created or extended.
9. Unity 6 shader validation report is clean, or the reason validation could not run is documented with the blocking Editor/process state.
10. No Unity .mat was created, assigned, inspected, or restored in this shader reconstruction goal.
```
