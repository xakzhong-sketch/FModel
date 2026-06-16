# Goal: UE Workspace Batch NoVisual Reconstruction

Use this goal from a material reconstruction workspace that contains multiple `*.bundle` directories.

This goal runs the no-visual reconstruction stage for multiple materials. It must not run RenderDoc analysis, screenshot capture, semantic GBuffer capture, lightweight smoke validation, or visual comparison.

## Inputs

```text
MatMap=MaterialMap.json
```

`MatMap=...` is optional. If it is omitted, default to `MaterialMap.json` in the current workspace root. Batch exports created from `UnityMatDir=...` or `UnityScene=...` should already contain this file, so the common invocation is:

```text
/goal UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md
```

Optional:

```text
UnityRoot=<UnityProjectRoot>
Filter=<bundle-name-or-substring>
ContinueOnError=true|false
Apply=true|false
UnityExe=<Unity Editor executable, manual override only>
```

Default `ContinueOnError` is `true`.

Running this Goal is explicit authorization for the NoVisual write set:

```text
shader/module generation
static module/function coverage audit
material restore DryRun/report generation
Unity shader import/compile check
validation helper install/update
per-bundle handoff/report generation
```

It does not authorize `.mat` Apply, visual validation, RenderDoc analysis, screenshot capture, semantic GBuffer capture, lightweight smoke validation, FModel/CUE4Parse source edits, or Doc template edits.

`UnityRoot=...` is only required when `MaterialMap.json` does not contain a valid local Unity project root, or when the workspace was moved to another machine. The compile check must read `ProjectSettings/ProjectVersion.txt` from that Unity project and use the matching Unity editor version. Do not pick a different Unity version just because it exists on the machine. `UnityExe=...` is a manual override/fallback, not the normal path.

The common first pass is DryRun-only and does not replace shaders on Unity `.mat` files:

```text
/goal UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md
```

After the first pass reports clean compile/import status, the explicit Apply pass is:

```text
/goal UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md Apply
```

Apply mode writes the mapped Unity `.mat` files and updates their shader references/material values. It must be skipped for any bundle whose shader compile/import evidence is missing or failed, whose DryRun reports unresolved `MissingTextureGuids`, or whose shader reassignment would skip missing Unity material properties.

## First Read

1. `AGENTS.md`
2. `batch_manifest.json`
3. `summary.json`
4. `MaterialMap.json`

If `MaterialMap.json` is missing, inspect `batch_manifest.json` and `summary.json` only to explain what is missing. Do not guess bundle-to-material mappings. Ask for `MatMap=...` or rerun batch export with `UnityMatDir=...` / `UnityScene=...`.

## MaterialMap Schema

```json
{
  "unityRoot": "<UnityProjectRoot>",
  "items": [
    {
      "bundle": "MI_Target.bundle",
      "mat": "Assets/.../MI_Target.mat",
      "materialName": "MI_Target",
      "unrealMaterialPath": "/Game/.../MI_Target",
      "tags": []
    }
  ]
}
```

Rules:

- `bundle` is required and must point to a direct `*.bundle` child unless it is an absolute bundle path.
- `mat` is required and should be an `Assets/.../*.mat` relative path when the map was generated from a Unity `.mat` directory.
- `unrealMaterialPath` is optional for NoVisual restore, but generated UnityMatDir maps should include it for auditability.
- `unityRoot` may be in the map or supplied as `UnityRoot=...`. If the map contains `<UnityProjectRoot>`, the Agent must ask for or use an explicit `UnityRoot=...` before writing Unity files.
- For UnityMatDir-generated maps, `Assets/.../MI.mat` corresponds only to the same relative UE path `/Game/.../MI`. Do not substitute another cooked UE material just because it has the same asset name under a different folder.
- Duplicate bundle entries are errors.
- Duplicate `.mat` paths should be reported as warnings before any writes.

## Scope Rules

- Serialize all Unity project writes.
- Do not spawn multiple write-capable sub-agents against the same Unity `Assets` tree.
- Allowed Unity writes are limited to generated/reused shader files, validation helper files, explicit material restore targets, and explicit missing-texture output directories such as `TextureOut=Assets/...`.
- Do not modify FModel/CUE4Parse exporter source, Doc templates, or other tool repositories while executing this reconstruction/export workflow.
- Do not run `UE_Unity_Material_Semantic_Visual_Validation_Goal.md`.
- Do not run `UE_Unity_Material_Lightweight_Visual_Smoke_Goal.md`.
- Do not run `UE_RenderDoc_Compact_Summary_Goal.md`.
- Do not read `VisualRefs`, screenshots, or raw RenderDoc data.
- A bundle item cannot be marked successful if Unity shader import/compile has errors or lacks explicit compile/import success evidence.

Compile/import success evidence means a structured Unity report such as:

```json
{
  "Schema": "sn2-unity-shader-validation/v1",
  "Ok": true,
  "ErrorCount": 0
}
```

Do not treat log text containing words like `import` or `compiled` as success unless the structured report has no errors.

## Helper Script

The repository helper is:

```powershell
python <FModelRepo>\CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\workspace_batch_no_visual.py `
  --workspace . `
  --run-generator `
  --run-restore-dryrun `
  --run-compile-check `
  --add-missing `
  --continue-on-error `
  --require-compile-success
```

Default Goal mode runs the full NoVisual stage: it validates workspace `MaterialMap.json`, verifies bundles, generates or reuses Unity shaders/modules, audits static module/function coverage from cooked evidence, runs material restore DryRun with missing Unity properties included in the report, imports missing textures from bundle/shared texture payloads when available, refreshes Unity so `.meta` files are generated, reruns DryRun, installs/updates the validation helper, runs Unity shader import/compile checks, writes per-bundle handoff summaries, and writes the workspace batch report. It does not apply `.mat` writes, capture screenshots, run RenderDoc, or run visual validation.

Advanced helper flags:

```text
--run-generator
--overwrite-generated
--run-restore-dryrun
--run-compile-check
--unity-exe <Unity.exe>
--apply --apply-confirm WRITE_MAT
--add-missing
--no-add-missing
--skip-texture-payload-import
--texture-out <Assets/...>
--overwrite-textures
--allow-scaffold-success
--continue-on-error
--filter <text>
```

Rules:

- Use `--run-restore-dryrun` to produce/update `analysis/unity_material_restore_report.json` without writing `.mat`.
- Use `--run-compile-check` to run Unity batchmode `SN2ShaderValidation.Run` and write `analysis/unity_shader_compile_report.json`.
  The helper must read the Unity editor version from `<UnityRoot>/ProjectSettings/ProjectVersion.txt` and use that matching editor install. It must install `CUE4Parse/CUE4Parse.ShaderBundleExporter/Tools/unity_shader_validation/SN2ShaderValidation.cs.txt` into `Assets/Editor/ShaderReverse/Validation/SN2ShaderValidation.cs` if the Unity project does not already have the current runner. Do not hand-author a replacement validation script during batch reconstruction.
- Use `--unity-exe <Unity.exe>` only as an explicit manual fallback when the matching editor cannot be found from the Unity project version.
- Batch material restore passes `--add-missing` by default so newly generated shader properties are actually written to the `.mat`. Use `--no-add-missing` only for audit/debugging.
- If DryRun reports `MissingTextureGuids` and the bundle has `texture_payload/manifest.json`, `texture_payload_manifest.json`, or workspace shared `Textures/manifest.json`, the helper should automatically run `import_bundle_texture_payload.py`, copy only the missing payload textures to `--texture-out` (default `Assets/Art/Recovered/Subnautica2`), refresh Unity to generate `.meta`, and rerun DryRun before Apply.
- Use `--skip-texture-payload-import` only when the user wants to inspect missing texture reports without writing imported texture files.
- Use `--overwrite-textures` only when replacing already imported payload files is intended.
- Use `--apply --apply-confirm WRITE_MAT` only when the current `/goal` invocation explicitly asks to apply material values. The helper must not apply a `.mat` unless compile/import evidence is successful and texture GUIDs are resolved. If `MissingTextureGuids` or `TextureExportNeeded=true` appears in the DryRun report, export/import those textures and wait for Unity `.meta` files before Apply.
- Use `--allow-scaffold-success` only for an intentional low-fidelity batch scaffold pass. It allows static-incomplete items to count as success, but the report must still show `StaticReconstructionCoverageStatus`.
- Keep Unity writes serialized; do not run multiple helper instances against the same Unity project.
- The helper writes command logs under `<Bundle>/analysis/batch_no_visual/` with command, exit code, stdout bytes, and stderr bytes.

## Static Reconstruction Bar

NoVisual is not a compile-only scaffold. It does not prove final visual parity, but it must still cover static cooked evidence before a bundle can be called successful.

Before declaring success, inspect:

```text
analysis/unity_layer_reconstruction_contract.json
analysis/material_layer_stack.json
analysis/material_layer_parameter_bindings.json
analysis/material_static_permutation.json
analysis/material_function_dependencies.json
analysis/module_formula_evidence.json
analysis/dxil_formula_evidence.json
analysis/curve_atlas_metadata.json
source/material.cooked.json
source/material_functions/*.cooked.json
```

For every evidenced Master/Layer/Blend/MaterialFunction/static feature, the output must classify it as:

```text
implemented
approximated_with_reason
deferred_with_missing_evidence
renderer_only_or_unity_substitute
```

The helper writes this classification summary to:

```text
<Bundle>/analysis/unity_no_visual_reconstruction_summary.json
  StaticReconstructionCoverage
  StaticReconstructionCoverageStatus
```

If cooked evidence names visible features such as `MB_MaskID` height-aware blending, `MB_VertexColorOverlay`, `MaterialExpressionVertexColor`, `ML_LayerCustomPrim_CurveGradient`, `MF_Project_CPD_Packed`, CPD parameters, `NormalFromHeightMap`, curve/gradient atlas data, or `bHasWorldPosition`, the Agent must not silently ignore them. Implement them from available static evidence where possible. If they cannot be implemented without visual/runtime evidence, mark the bundle `needs_input` / `needs_static_reconstruction` and write the exact missing evidence and expected visual impact.

Do not mark a bundle `success` just because:

```text
the shader compiles
Properties exist
textures bind
.mat DryRun or Apply succeeded
```

Those are necessary gates, not sufficient reconstruction completion.

## Per-Bundle Procedure

For each mapped item, run the equivalent of:

```text
/goal <Bundle>\UE_Unity_Material_OneClick_NoVisual_Reconstruction_Goal.md -mat <Mat>
```

The bundle-local NoVisual goal is authoritative for the material reconstruction steps. This workspace goal only coordinates multiple bundles and enforces serialization/no-visual boundaries.

For each bundle:

1. Verify the bundle with `commands/verify_bundle.ps1`.
2. Read compact cooked evidence first:
   - `analysis/ai_context_pack.md`
   - `analysis/reconstruction_entrypoints.json`
   - `analysis/unity_layer_reconstruction_contract.json`
   - `analysis/material_layer_stack.json`
   - `analysis/material_layer_parameter_bindings.json`
   - `analysis/unity_shader_assignment.json`
3. Reconstruct or reuse the Unity shader.
   Treat the shader reuse decision as provisional until semantic visual validation or stronger runtime/DXIL evidence confirms it. If later validation invalidates `reuse_existing`, upgrade the follow-up path to `extend_existing` or `create_new` and record evidence/regression risk instead of broad-patching a shared shader for one MI.
4. Produce a static reconstruction coverage summary. If coverage is `needs_static_reconstruction`, continue writing reports for handoff but do not call the bundle successful.
5. Run material restore DryRun.
6. Resolve missing texture GUIDs through:
   - bundle-local `texture_payload/manifest.json`;
   - shared workspace `Textures/manifest.json` with bundle `texture_payload_manifest.json`;
   - existing Unity project texture assets;
   - explicit missing texture export from original game data.
   The batch helper must import available bundle/shared payloads automatically before asking for original cooked game data.
7. Apply material values only after DryRun is clean, all required texture `.meta` GUIDs are resolved, static coverage is not blocking, and the current goal invocation explicitly permits Apply.
8. Prove Unity shader import/compile succeeds.
9. Write per-bundle handoff reports.

If an Agent intentionally runs the helper without the NoVisual automation flags for audit/debugging, it may only mark a bundle `success` if the required compile/import and restore reports already exist and are successful. Otherwise it must mark the bundle `needs_input` or `failed`.

## Required Per-Bundle Outputs

```text
<Bundle>/analysis/unity_no_visual_reconstruction_summary.json
<Bundle>/analysis/unity_no_visual_reconstruction_summary.md
<Bundle>/analysis/unity_shader_compile_report.json
<Bundle>/analysis/unity_material_restore_report.json
<Bundle>/analysis/unity_handoff_for_visual_validation.md
<Bundle>/analysis/batch_no_visual/*.command.json
```

`unity_handoff_for_visual_validation.md` must include:

- bundle path;
- UE material path;
- Unity `.mat` path;
- generated or reused Unity shader path;
- shader reuse decision;
- static reconstruction coverage status;
- implemented/approximated/deferred module and feature list;
- compile/import result;
- restored parameter count;
- unresolved parameters;
- missing/unresolved textures;
- texture payload source: bundle, shared, Unity project, original game data, or missing;
- implemented Master/Layer/Blend modules;
- approximate or omitted features;
- whether RenderDoc data exists;
- expected RenderDoc capture location: `<Bundle>/RenderDocCapture`;
- recommended next validation goal.

## Workspace Outputs

```text
batch_reports/no_visual_batch_report.json
batch_reports/no_visual_batch_report.md
```

The workspace report should list each item as one of:

```text
success
failed
skipped
needs_input
```

For each item, include:

- bundle;
- `.mat`;
- status;
- shader path;
- compile/import status;
- material restore status;
- texture payload mode;
- handoff summary path;
- important errors.
- command log paths, exit code, stdout byte count, and stderr byte count for commands the helper actually ran.

Do not inline large logs. Store paths, exit codes, and stdout/stderr byte counts.

## Multi-Bundle Goal Invocation Rules

In a multi-bundle workspace, do not use bare bundle-local goal filenames such as:

```text
/goal UE_Unity_Material_OneClick_NoVisual_Reconstruction_Goal.md
```

Use the workspace batch goal:

```text
/goal UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md MatMap=MaterialMap.json
```

For the generated-map path, this shorter form is preferred:

```text
/goal UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md
```

Or an explicit bundle-local path:

```text
/goal MI_Target.bundle\UE_Unity_Material_OneClick_NoVisual_Reconstruction_Goal.md -mat <Mat>
```

If a request is ambiguous, stop and ask for `Bundle=...`.

## Completion Criteria

This goal is complete only when:

1. Every selected MaterialMap item has a status in the workspace batch report.
2. Every successful item has shader compile/import success evidence.
3. Every successful item has all required per-bundle handoff files.
4. No RenderDoc, screenshot, semantic capture, lightweight smoke, or visual comparison step was run.
5. Failed or skipped items are explicitly listed with actionable reasons.
