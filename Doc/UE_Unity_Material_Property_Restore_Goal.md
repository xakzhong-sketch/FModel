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

/Goal D:\Github\FModel\Doc\UE_Unity_Material_Property_Restore_Goal.md Mat=K:\Project\Assets\...\MI_Name.mat ExportMissingTextures
```

Hard safety rule:

```text
Apply Token Gate:
  Only run a command with --apply when the user's current /Goal invocation contains the standalone token Apply.
  Any --apply command must also pass --apply-confirm WRITE_MAT; otherwise the script must fail.
  Mat=... alone is always DryRun.
  "restore", "还原", "恢复", "处理", "fix", or "update" must not be interpreted as Apply.
  A clean DryRun report must not automatically trigger Apply.
  After DryRun, stop and report the dry-run result. The user must start a new explicit Apply invocation before any .mat write.
```

Argument rules:

```text
Mat=...
  Unity .mat file to update.

Bundle=...
  Bundle directory containing parameters/material_parameters.json and parameters/textures.json.

Apply
  Actually write the .mat file. Do not apply before a dry-run report looks correct.
  This is valid only when the user's current /Goal invocation includes the standalone token Apply.
  The Python command must include both --apply and --apply-confirm WRITE_MAT.
  Do not infer Apply from Mat=..., from the word restore/还原/恢复, or from a successful DryRun.

DryRun
  Default mode. Generates a report only and does not modify the .mat file.
  If neither Apply nor ExportMissingTextures is present, this is the only allowed mode.

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
  Never use CreateIfMissing during a Mat=... only invocation.

ShaderMeta=...
  Optional Unity .shader.meta file for the assigned/reused shader.
  Use this when the .mat still points at an older shader and must update m_Shader by GUID.

ShaderGuid=...
  Optional direct Unity shader GUID. Prefer ShaderMeta when the .meta file is available.

ExportMissingTextures
  Optional restore-stage sub-step. If DryRun reports MissingTextureGuids, export only those missing UE cooked textures listed by MissingTextureExportCandidates.
  This must not be combined with Apply and must not modify the .mat file.

Texture payload import
  If DryRun reports MissingTextureGuids and Bundle\texture_payload\manifest.json exists, import only the missing textures from that payload before using ExportMissingTextures.
  Payload import does not require original UE cooked game data and must not modify the .mat file.

TextureOut=...
  Optional Unity Assets-relative output folder for ExportMissingTextures.
  Default expectation: Assets/Art/Recovered/Subnautica2

RestoreReport=...
  Optional existing dry-run report for ExportMissingTextures.
  If omitted, run DryRun first and use the newly generated report.
```

If invoked from a material workspace directory:

```text
1. Record the starting directory as ROOT_DIR.
2. If ROOT_DIR contains exactly one *.bundle directory, use it as Bundle.
3. If ROOT_DIR contains exactly one *.mat file, use it as Mat.
4. If multiple bundles or mats exist, ask the user which one to use.
```

When `ExportMissingTextures` is requested from a workspace directory:

```text
1. Resolve Bundle from the current directory using the same rules above.
2. Run or locate a DryRun restore report for Mat + Bundle.
3. Read MissingTextureExportCandidates from that report.
4. Use Source.Game, Source.Paks, and Source.Mapping from Bundle\manifest.json.
5. Call CUE4Parse.ShaderBundleExporter --export-missing-unity-textures.
6. Do not pass --apply to unity_material_apply_ue_params.py in this sub-step.
```

Missing texture recovery source order:

```text
1. Existing Unity texture assets with .meta GUIDs under AssetsRoot / TextureSearchRoot.
2. Bundle texture_payload/manifest.json, if present.
3. Original UE cooked game data through explicit ExportMissingTextures.
4. If none are available, stop and report that missing Unity textures cannot be recovered from this bundle alone.
```

When texture payload exists and DryRun reports `MissingTextureGuids`:

```text
1. Read Bundle\texture_payload\manifest.json.
2. Copy only textures needed by MissingTextureExportCandidates into Unity Assets.
3. Copy sidecar JSON files with those textures.
4. Do not modify the .mat file.
5. Let Unity import textures and generate .meta files.
6. Rerun DryRun.
7. Apply only after MissingTextureGuids is empty or remaining missing textures are explicitly accepted.
```

When neither `Apply` nor `ExportMissingTextures` is present:

```text
1. Run exactly one DryRun restore command.
2. The Python command must not include --apply, --create-if-missing, or --no-backup.
3. Write the JSON report.
4. Stop after reporting the result.
5. Do not proceed to Apply even if MissingTextureGuids=0, SkippedMissingUnityProperties=0, and Collisions is empty.
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

Only use this command when the user explicitly invoked this Goal with `Apply` and requested creation of a missing material.

```powershell
python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_material_apply_ue_params.py `
  --mat "UNITY_MAT" `
  --bundle-dir "BUNDLE_DIR" `
  --layer-aware `
  --assign-shader-meta "ASSIGNED_SHADER.shader.meta" `
  --add-missing `
  --create-if-missing `
  --apply `
  --apply-confirm WRITE_MAT `
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

This is the required command category for `/Goal ... Mat=...` when `Apply` is absent. Do not append `--apply`.

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
Dry run only. Re-run with --apply --apply-confirm WRITE_MAT to write the .mat file.
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

TextureExportNeeded
  True when the dry-run report contains missing texture export candidates.

MissingTextureExportCandidates
  Deduplicated UE cooked textures that can be exported from the original game data.
  These are generated only for unresolved Unity texture GUIDs and contain ObjectPath, TextureName, ImportIntent, ColorSpace, and suggested Unity output path.

TextureExportCommandHint
  Suggested C# exporter command. Treat this as a command shape; verify paths before running.

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

The user must explicitly invoke:

```text
/Goal D:\Github\FModel\Doc\UE_Unity_Material_Property_Restore_Goal.md Apply Mat=... Bundle=...
```

If the current invocation did not contain `Apply`, do not run this command.

```powershell
python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_material_apply_ue_params.py `
  --mat "UNITY_MAT" `
  --bundle-dir "BUNDLE_DIR" `
  --layer-aware `
  --assign-shader-meta "ASSIGNED_SHADER.shader.meta" `
  --report-out "REPORT_OUT" `
  --apply `
  --apply-confirm WRITE_MAT
```

By default the script creates a timestamped backup next to the `.mat`:

```text
MI_Name.mat.bak_YYYYMMDD_HHMMSS
```

Use `--no-backup` only for disposable test files.

## After Apply: Visual Validation

After a successful `Apply`, do not claim visual fidelity from the restore report alone.

If the current material workspace contains original-game reference screenshots under `VisualRefs` / `ScreenShot*`, or the user requests reference-based visual validation, run:

```text
/Goal UE_Unity_Material_Semantic_Visual_Validation_Goal.md
```

If there are no reference screenshots but the user has opened the Unity scene containing the target object and centered it in GameView, run:

```text
/Goal UE_Unity_Material_Lightweight_Visual_Smoke_Goal.md
```

Use the bundle-local goal files when they exist. Use `<FModelRepo>\Doc\UE_Unity_Material_Semantic_Visual_Validation_Goal.md` or `<FModelRepo>\Doc\UE_Unity_Material_Lightweight_Visual_Smoke_Goal.md` as canonical fallbacks only.

Semantic validation captures Unity semantic outputs and compares compact reports before raw screenshots:

```text
UnityValidation\reports\visual_validation_report.md
UnityValidation\reports\visual_validation_report.json
UnityValidation\reports\visual_validation_advice.json
```

Rules:

```text
1. The visual validation goal must not modify `.mat` files.
2. Read semantic report/advice before inspecting raw captures, diffs, or final lit screenshots.
3. Prefer decoded semantic GBuffer/material channel captures such as albedo.png, normal_world.png, metallic.png, smoothness.png, and occlusion.png. raw_gbuffer0/1/2.png are debugging evidence only.
4. A final lit screenshot alone is not proof that BaseColor, Normal, Roughness/Smoothness, Metallic/AO, Alpha/Mask, LayerBlend, or HeightBlend are correct.
5. Lightweight smoke validation is no-reference validation. It may fail obvious rendering errors or obvious contradictions between implemented shader features and current Unity rendering, but it must not claim UE visual parity.
```

## Optional: Export Missing Textures

Use this only after DryRun reports `MissingTextureGuids` and the report contains `MissingTextureExportCandidates`.

Do not run this during `PROMPT_NEXT_SESSION.md` shader reconstruction. Do not combine it with `Apply`.

If the bundle contains `texture_payload/manifest.json`, prefer payload import first:

```powershell
python <FModelRepo>\CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\import_bundle_texture_payload.py `
  --bundle "BUNDLE_DIR" `
  --restore-report "DRYRUN_REPORT.json" `
  --unity-assets-root "<UnityProject>\Assets" `
  --texture-out "Assets/Art/Recovered/Subnautica2"
```

Payload import output:

```text
<Bundle>\unity_texture_payload_import_report.json
<UnityAssetsRoot>\<TextureOut>\Game\...\TextureName.png
<UnityAssetsRoot>\<TextureOut>\Game\...\TextureName.png.ue_texture_export.json
```

Only use the cooked-data export command below when the bundle has no payload, when the payload is incomplete, or when the user explicitly wants to regenerate textures from original UE cooked data.

Command shape:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --export-missing-unity-textures `
  --game "GAME_FROM_BUNDLE_MANIFEST" `
  --paks "PAKS_FROM_BUNDLE_MANIFEST" `
  --mapping "MAPPING_FROM_BUNDLE_MANIFEST" `
  --bundle "BUNDLE_DIR" `
  --restore-report "DRYRUN_REPORT.json" `
  --unity-assets-root "K:\Project\Assets" `
  --texture-out "Assets/Art/Recovered/Subnautica2"
```

Output:

```text
<Bundle>\unity_missing_texture_export_report.json
<UnityAssetsRoot>\<TextureOut>\Game\...\TextureName.png
<UnityAssetsRoot>\<TextureOut>\Game\...\TextureName.png.ue_texture_export.json
```

Rules:

```text
1. Export only textures listed in MissingTextureExportCandidates.
2. Existing texture files are skipped unless --overwrite-textures is explicit.
3. The command never modifies Unity .mat files.
4. Sidecar JSON records UE ObjectPath, ImportIntent, ColorSpace, and property usage.
5. After Unity imports the exported textures and creates .meta files, rerun DryRun.
6. Apply only after MissingTextureGuids is empty or the remaining missing textures are explicitly accepted.
```

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
5. If MissingTextureGuids was not accepted, ExportMissingTextures was run, Unity imported the exported files, and a second DryRun can resolve the new texture GUIDs.
6. SkippedMissingUnityProperties is reviewed and either fixed in the Unity shader/material or accepted as intentionally omitted.
7. If Apply was requested, the .mat is updated and a .bak file exists unless --no-backup was explicitly used.
8. Final response reports Mat, Bundle, report path, matched count, StableKey count, legacy fallback count, skipped count, missing texture GUID count, texture export count if used, and whether Apply was used.
9. If Apply was used, final response tells the user which post-restore validation is appropriate: local `UE_Unity_Material_Semantic_Visual_Validation_Goal.md` when `VisualRefs` / `ScreenShot*` references exist, or local `UE_Unity_Material_Lightweight_Visual_Smoke_Goal.md` when no references exist but the user has prepared the target scene in GameView.
```
