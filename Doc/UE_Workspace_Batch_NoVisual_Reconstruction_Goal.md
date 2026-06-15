# Goal: UE Workspace Batch NoVisual Reconstruction

Use this goal from a material reconstruction workspace that contains multiple `*.bundle` directories.

This goal runs the no-visual reconstruction stage for multiple materials. It must not run RenderDoc analysis, screenshot capture, semantic GBuffer capture, lightweight smoke validation, or visual comparison.

## Inputs

```text
MatMap=MaterialMap.json
```

`MatMap=...` is optional. If it is omitted, default to `MaterialMap.json` in the current workspace root. Batch exports created from `UnityMatDir=...` should already contain this file, so the common invocation is:

```text
/goal UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md
```

Optional:

```text
UnityRoot=<UnityProjectRoot>
Filter=<bundle-name-or-substring>
ContinueOnError=true|false
RunGenerator=true|false
RunRestoreDryRun=true|false
RunCompileCheck=true|false
UnityExe=<Unity Editor executable>
Apply=true|false
```

Default `ContinueOnError` is `false`.
Default write behavior is read-only audit/handoff. Shader generation, material restore, and `.mat` writes require explicit user intent in the current `/goal` invocation.

## First Read

1. `AGENTS.md`
2. `batch_manifest.json`
3. `summary.json`
4. `MaterialMap.json`

If `MaterialMap.json` is missing, inspect `batch_manifest.json` and `summary.json` only to explain what is missing. Do not guess bundle-to-material mappings. Ask for `MatMap=...` or rerun batch export with `UnityMatDir=...`.

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
  --require-compile-success
```

Default helper mode is read-only audit: it validates workspace `MaterialMap.json`, inspects existing per-bundle NoVisual artifacts, writes/refreshes handoff summaries, and writes the workspace batch report. It does not generate shaders, restore `.mat` files, capture screenshots, run RenderDoc, or run visual validation.

Explicit automation flags:

```text
--run-generator
--overwrite-generated
--run-restore-dryrun
--run-compile-check --unity-exe <Unity.exe>
--apply --apply-confirm WRITE_MAT
--add-missing
--continue-on-error
--filter <text>
```

Rules:

- Use `--run-generator` only when the goal is allowed to write generated Unity shader/module files.
- Use `--run-restore-dryrun` to produce/update `analysis/unity_material_restore_report.json` without writing `.mat`.
- Use `--run-compile-check --unity-exe <Unity.exe>` to run Unity batchmode `SN2ShaderValidation.Run` and write `analysis/unity_shader_compile_report.json`.
- Use `--apply --apply-confirm WRITE_MAT` only when the current `/goal` invocation explicitly asks to apply material values.
- Keep Unity writes serialized; do not run multiple helper instances against the same Unity project.
- The helper writes command logs under `<Bundle>/analysis/batch_no_visual/` with command, exit code, stdout bytes, and stderr bytes.

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
4. Run material restore DryRun.
5. Resolve missing texture GUIDs through:
   - bundle-local `texture_payload/manifest.json`;
   - shared workspace `Textures/manifest.json` with bundle `texture_payload_manifest.json`;
   - existing Unity project texture assets;
   - explicit missing texture export from original game data.
6. Apply material values only after DryRun is clean and the current goal invocation explicitly permits Apply.
7. Prove Unity shader import/compile succeeds.
8. Write per-bundle handoff reports.

When the helper is used in read-only audit mode, it may only mark a bundle `success` if the required compile/import and restore reports already exist and are successful. Otherwise it must mark the bundle `needs_input` or `failed`.

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
