# Goal: Restore Unity Material Properties From UE Cooked Bundle

Objective: apply original UE cooked material parameter values from a shader bundle to an existing Unity `.mat` file after the Unity shader and material have already been created.

This goal does not reconstruct shader code. It only restores saved Unity material values from:

```text
<Bundle>\analysis\material_layer_parameter_bindings.json, when present
<Bundle>\parameters\material_parameters.json
<Bundle>\parameters\textures.json
```

## How To Invoke This Goal

Preferred forms:

```text
/Goal D:\Github\FModel\Doc\UE_Unity_Material_Property_Restore_Goal.md Mat=K:\Project\Assets\...\MI_Name.mat Bundle=K:\WorkSpace\ShaderReverse\MI_Name\MI_Name.bundle

/Goal D:\Github\FModel\Doc\UE_Unity_Material_Property_Restore_Goal.md Apply Mat=K:\Project\Assets\...\MI_Name.mat Bundle=D:\ShaderWP\MI_Name.bundle

/Goal D:\Github\FModel\Doc\UE_Unity_Material_Property_Restore_Goal.md DryRun current directory
```

Argument rules:

```text
Mat=...
  Unity .mat file to update.

Bundle=...
  Bundle directory containing parameters/material_parameters.json and parameters/textures.json.

Apply
  Actually write the .mat file. Do not apply before a dry-run report looks correct.

DryRun
  Default mode. Generates a report only and does not modify the .mat file.

AssetsRoot=...
  Optional search root for Unity texture .meta files. If omitted, use the nearest parent named Assets from Mat.

TextureSearchRoot=...
  Optional additional texture .meta search root. Can be repeated when running the script manually.

LayerAware
  Use this when the bundle contains analysis/material_layer_parameter_bindings.json.
  This is the default expectation for Material Layer / Material Blend bundles.

CreateIfMissing
  Optional. Create a new Unity .mat when Mat does not exist.
  Requires Apply plus ShaderMeta or ShaderGuid.
  Use this when shader reuse says an MI should share an existing shader but no Unity material file exists yet.

ShaderMeta=...
  Optional Unity .shader.meta file for the assigned/reused shader.
  Use this when the .mat still points at an older shader and must update m_Shader by GUID.

ShaderGuid=...
  Optional direct Unity shader GUID. Prefer ShaderMeta when the .meta file is available.
```

If invoked from a material workspace directory:

```text
1. Record the starting directory as ROOT_DIR.
2. If ROOT_DIR contains exactly one *.bundle directory, use it as Bundle.
3. If ROOT_DIR contains exactly one *.mat file, use it as Mat.
4. If multiple bundles or mats exist, ask the user which one to use.
```

## Preconditions

The Unity shader should already exist and should preserve UE material properties one-to-one.

Required Unity shader/material behavior:

```text
1. The Unity Shader Properties block preserves UE parameter names, types, defaults, and texture references from the bundle.
2. For Material Layer bundles, Unity internal property names are StableKey-derived, for example _Layer5_Gradient_Curve or _Blend0_Color_Mask_Input.
3. Same-name UE parameters are not merged when Association or Index differs.
4. The Unity .mat uses that shader.
5. The Unity .mat name matches the source UE material instance when possible.
6. Texture assets have Unity .meta files with GUIDs.
```

If the `.mat` does not yet use the assigned shader, pass `--assign-shader-meta` or `--assign-shader-guid` during dry-run and apply. Shader assignment is reported separately from parameter restoration.

The restore script defaults to updating existing Unity properties only. Missing properties are reported instead of silently appended. Use `--add-missing` only when the generated Unity shader intentionally defines those properties but the `.mat` file has not serialized them yet.

For a new MI that reuses an existing shader, create the material with:

```powershell
python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_material_apply_ue_params.py `
  --mat "UNITY_MAT" `
  --bundle-dir "BUNDLE_DIR" `
  --layer-aware `
  --assign-shader-meta "ASSIGNED_SHADER.shader.meta" `
  --add-missing `
  --create-if-missing `
  --apply `
  --report-out "REPORT_OUT"
```

## Script

Run from:

```powershell
cd D:\Github\FModel
```

Script path:

```text
CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_material_apply_ue_params.py
```

## Dry Run Command

Replace `UNITY_MAT`, `BUNDLE_DIR`, and `REPORT_OUT`:

```powershell
python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_material_apply_ue_params.py `
  --mat "UNITY_MAT" `
  --bundle-dir "BUNDLE_DIR" `
  --layer-aware `
  --assign-shader-meta "ASSIGNED_SHADER.shader.meta" `
  --report-out "REPORT_OUT"
```

Optional faster texture GUID search:

```powershell
python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_material_apply_ue_params.py `
  --mat "UNITY_MAT" `
  --bundle-dir "BUNDLE_DIR" `
  --assets-root "K:\Project\Assets\Art\Surfaces\SpecificTextureFolder" `
  --layer-aware `
  --report-out "REPORT_OUT"
```

Dry-run output should say:

```text
Dry run only. Re-run with --apply to write the .mat file.
Report: ...
Matched: N, Added: 0, MissingTextureGuids: 0
```

Layer-aware output also reports:

```text
StableKey: N
LegacyFallback: N
```

For a correctly generated layer-aware Unity shader, `StableKey` should cover the expected core/layer/blend properties. `LegacyFallback` means the Unity `.mat` is still using old property names or the shader has not adopted StableKey-derived names.

## Review Report

Open the report JSON before applying.

Important fields:

```text
MatchedProperties
  Existing Unity properties that will be changed.

MatchedByStableKey
  Existing Unity properties matched through analysis/material_layer_parameter_bindings.json StableKey -> UnityName mappings.

MatchedByLegacyNameFallback
  Matches that only succeeded through old name/sanitized-name candidates. For layer-aware shaders this should be treated as a shader property naming mismatch unless intentionally supported.

Collisions
  Generated layer-aware UnityName collisions. This must be empty before applying.

AssignedUnityShader / ShaderAssignmentDecision
  Echoes analysis/unity_shader_assignment.json so the restore report records whether this MI should reuse a shared shader.

ShaderReferenceUpdate
  Present when --assign-shader-meta or --assign-shader-guid is used.
  Check PreviousGuid, Guid, Changed, PreviousLine, and NewLine before applying.

SkippedMissingUnityProperties
  UE parameters that were not found in the Unity .mat. Treat these as shader/material generation mismatches unless intentionally omitted.

MissingTextureGuids
  Texture parameters whose Unity .meta GUID could not be resolved. Fix texture imports/search roots before applying.

AddMissing
  Must be false for the normal restore flow.

MaterialWasCreated
  True when --create-if-missing generated a new Unity .mat before applying properties.
```

For Scalar values, the script prefers the exported final-value map:

```text
parameters/material_parameters.json -> ScalarParameters
```

This matters because `NumericParameters` can contain repeated names from multiple cooked material resources.

## Apply Command

Only apply after the dry-run report is correct:

```powershell
python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_material_apply_ue_params.py `
  --mat "UNITY_MAT" `
  --bundle-dir "BUNDLE_DIR" `
  --layer-aware `
  --assign-shader-meta "ASSIGNED_SHADER.shader.meta" `
  --report-out "REPORT_OUT" `
  --apply
```

By default the script creates a timestamped backup next to the `.mat`:

```text
MI_Name.mat.bak_YYYYMMDD_HHMMSS
```

Use `--no-backup` only for disposable test files.

## Test Example

Example tested with `MI_CG_RockSmooth_01a`:

```powershell
python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_material_apply_ue_params.py `
  --mat "K:\WorkSpace\trunk\ExportedProject\Assets\Art\Environment\Biome\CoralGarden\Rocks\Material\MI_CG_RockSmooth_01a.mat" `
  --bundle-dir "D:\Tmp\ShaderBundles\MI_CG_RockSmooth_01a.bundle" `
  --assets-root "K:\WorkSpace\trunk\ExportedProject\Assets\Art\Surfaces\Tiling\CoralGarden\Rock\CG_RockSmooth_01a" `
  --layer-aware `
  --assign-shader-meta "K:\WorkSpace\trunk\ExportedProject\Assets\Shaders\Subnautica2\Reconstructed\M_LayerStandard\LayerStack_C43830F3.shader.meta" `
  --report-out "D:\Tmp\ShaderBundles\MI_CG_RockSmooth_01a.unity_material_restore_report.json"
```

Observed dry-run result:

```text
Matched: 11
StableKey: 0
LegacyFallback: 11
Added: 0
MissingTextureGuids: 0
ShaderChanged: true/false
```

Interpretation:

```text
The tested .mat still used an older non-StableKey shader property schema, so --layer-aware correctly fell back to legacy names.
After the Unity shader is regenerated with _LayerN_ / _BlendN_ StableKey property names, MatchedByStableKey should increase and LegacyFallback should drop.
```

## Completion Criteria

The task is complete only when:

```text
1. Dry-run report exists and is parseable JSON.
2. MatchedProperties contains the expected UE-to-Unity property mappings.
3. For Material Layer bundles, MatchedByStableKey covers the expected layer/blend/core properties and Collisions is empty.
4. MissingTextureGuids is empty, or unresolved textures are explicitly accepted.
5. SkippedMissingUnityProperties is reviewed and either fixed in the Unity shader/material or accepted as intentionally omitted.
6. If Apply was requested, the .mat is updated and a .bak file exists unless --no-backup was explicitly used.
7. Final response reports Mat, Bundle, report path, matched count, StableKey count, legacy fallback count, skipped count, missing texture GUID count, and whether Apply was used.
```
