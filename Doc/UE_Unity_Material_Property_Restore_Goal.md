# Goal: Restore Unity Material Properties From UE Cooked Bundle

Objective: apply original UE cooked material parameter values from a shader bundle to an existing Unity `.mat` file after the Unity shader and material have already been created.

This goal does not reconstruct shader code. It only restores saved Unity material values from:

```text
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
2. The Unity .mat uses that shader.
3. The Unity .mat name matches the source UE material instance when possible.
4. Texture assets have Unity .meta files with GUIDs.
```

The restore script defaults to updating existing Unity properties only. Missing properties are reported instead of silently appended. Use `--add-missing` only when the generated Unity shader intentionally defines those properties but the `.mat` file has not serialized them yet.

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
  --report-out "REPORT_OUT"
```

Optional faster texture GUID search:

```powershell
python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_material_apply_ue_params.py `
  --mat "UNITY_MAT" `
  --bundle-dir "BUNDLE_DIR" `
  --assets-root "K:\Project\Assets\Art\Surfaces\SpecificTextureFolder" `
  --report-out "REPORT_OUT"
```

Dry-run output should say:

```text
Dry run only. Re-run with --apply to write the .mat file.
Report: ...
Matched: N, Added: 0, MissingTextureGuids: 0
```

## Review Report

Open the report JSON before applying.

Important fields:

```text
MatchedProperties
  Existing Unity properties that will be changed.

SkippedMissingUnityProperties
  UE parameters that were not found in the Unity .mat. Treat these as shader/material generation mismatches unless intentionally omitted.

MissingTextureGuids
  Texture parameters whose Unity .meta GUID could not be resolved. Fix texture imports/search roots before applying.

AddMissing
  Must be false for the normal restore flow.
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
  --report-out "D:\Tmp\ShaderBundles\MI_CG_RockSmooth_01a.unity_material_restore_report.json"
```

Observed dry-run result:

```text
Matched: 15
Added: 0
MissingTextureGuids: 0
```

Key matched changes:

```text
Roughness Min -> _Roughness_Min: 0 -> 0.2
Roughness Power -> _Roughness_Power: 1 -> 1.25
BCM BaseColor|Metallic Map -> _BCM_BaseColor_Metallic_Map: T_CG_RockSmooth_01a_BCM
NRH Normal|Roughness|Height Map -> _NRH_Normal_Roughness_Height_Map: T_CG_RockSmooth_01a_NRH
```

## Completion Criteria

The task is complete only when:

```text
1. Dry-run report exists and is parseable JSON.
2. MatchedProperties contains the expected UE-to-Unity property mappings.
3. MissingTextureGuids is empty, or unresolved textures are explicitly accepted.
4. SkippedMissingUnityProperties is reviewed and either fixed in the Unity shader/material or accepted as intentionally omitted.
5. If Apply was requested, the .mat is updated and a .bak file exists unless --no-backup was explicitly used.
6. Final response reports Mat, Bundle, report path, matched count, skipped count, missing texture GUID count, and whether Apply was used.
```
