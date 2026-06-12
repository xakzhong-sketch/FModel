# UE Bundle Texture Payload Distribution Tasks

This task list implements `Doc/UE_Bundle_Texture_Payload_Distribution_Plan.md`.

## Status Legend

```text
todo       not started
doing      in progress
blocked    waiting on prerequisite or external input
done       implemented and verified
```

## Phase 0: Scope And Contract

### T0.1 Define Payload Contract

Status: done

Files:

```text
Doc/UE_Bundle_Texture_Payload_Distribution_Plan.md
Doc/UE_Unity_Material_Property_Restore_Goal.md
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/AgentWorkspaceWriter.cs
```

Tasks:

```text
1. State that source/textures/*.cooked.json is metadata only.
2. State that texture_payload/manifest.json is the optional Unity-importable image payload.
3. State that payload generation requires original UE cooked game data.
4. State that payload consumption must not require original UE cooked game data.
5. Keep PROMPT_NEXT_SESSION.md shader reconstruction texture-export-free.
```

Acceptance:

```text
Fresh Agent can distinguish metadata texture exports from decoded texture payload files.
Fresh Agent does not try to recover image pixels from source/textures/*.cooked.json.
```

### T0.2 Define Payload Manifest Schema

Status: done

Output:

```text
texture_payload/manifest.json
```

Tasks:

```text
1. Define Schema = ue-bundle-texture-payload/v1.
2. Include bundle name, material path, payload mode, generated time, and item list.
3. For each item include TextureName, ObjectPath, PayloadPath, SidecarPath, import intent, color space, dimensions, byte length, hash, and property references.
4. Include Missing and Warnings arrays.
5. Require bundle-relative paths only.
```

Acceptance:

```text
Manifest can be parsed independently from original game data.
Manifest contains no machine-local drive-rooted absolute paths.
```

## Phase 1: Documentation Integration

### T1.0 Add Texture Payload Export Goal Document

Status: done

New file:

```text
Doc/UE_Cooked_Material_Bundle_With_TexturePayload_Export_Goal.md
```

Purpose:

```text
Provide a second first-step export Goal for users who want a distributable bundle that includes Unity-importable decoded texture payload files.
```

Expected user-facing choice:

```text
Lightweight export:
  /goal <FModelRepo>/Doc/UE_Cooked_Material_Bundle_Export_Goal.md current directory export <MaterialName> material

Export with texture payload:
  /goal <FModelRepo>/Doc/UE_Cooked_Material_Bundle_With_TexturePayload_Export_Goal.md current directory export <MaterialName> material
```

Tasks:

```text
1. State that this Goal first performs the normal cooked material bundle export.
2. Then run --export-bundle-texture-payload --payload-scope referenced_textures.
3. Require original UE cooked game data on the exporting machine.
4. State that the output bundle is suitable for distribution to users who do not have original game paks.
5. State that generated texture payload is optional and size-sensitive.
6. State that shader reconstruction still starts from <Bundle>/PROMPT_NEXT_SESSION.md after export.
7. State that material restore can use texture_payload if Unity textures are missing.
8. Do not imply that this Goal is currently available until CLI support is implemented.
```

Acceptance:

```text
Users can choose between lightweight export and payload export without ambiguity.
Fresh Agents do not invent ad hoc scripts to add payload during the first export step.
```

### T1.1 Update Material Restore Goal Source Order

Status: done

File:

```text
Doc/UE_Unity_Material_Property_Restore_Goal.md
```

Tasks:

```text
1. Add source order:
   existing Unity .meta -> bundle texture_payload -> original UE cooked data -> blocker.
2. Document that payload import is separate from Apply.
3. Document that payload import copies only textures needed by the DryRun report.
4. Document that Unity import and second DryRun are required before Apply.
```

Acceptance:

```text
The restore goal tells an Agent what to do when original game data is unavailable but texture_payload exists.
Mat=... alone remains DryRun only.
```

### T1.2 Update Generated Bundle Docs

Status: done

File:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/AgentWorkspaceWriter.cs
```

Tasks:

```text
1. Add a bundle-local note explaining texture_payload.
2. Keep PROMPT_NEXT_SESSION.md shader/module-only.
3. Update one-click goals so texture payload is used only during material restore.
4. Add blocker wording for bundles without payload when original game data is unavailable.
```

Acceptance:

```text
Newly exported bundles contain Agent docs that explain both texture recovery paths.
No generated shader reconstruction prompt instructs Agents to export or import textures.
```

### T1.3 Update Human Workflow Docs

Status: done

Files:

```text
<HumanWorkflowRoot>/Workflow.md
<HumanWorkflowRoot>/Workflow_Simple.md
```

Tasks:

```text
1. Add a concise note that distributed bundles may include texture_payload.
2. Explain that if payload exists, missing textures can be imported without original game data.
3. Explain that if payload does not exist, ExportMissingTextures requires original UE cooked data.
```

Acceptance:

```text
Human docs stay short and do not become AI-only instructions.
```

## Phase 2: Payload Exporter

### T2.1 Add CLI Options

Status: done

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/ShaderBundleExportOptions.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Program.cs
```

New options:

```text
--export-bundle-texture-payload
--payload-scope referenced_textures | missing_from_restore_report
--payload-out <path>
--restore-report <DryRunReport.json>
--overwrite-textures
```

Tasks:

```text
1. Validate required provider inputs: --game, --paks, --mapping.
2. Validate --bundle.
3. Require --restore-report only for missing_from_restore_report scope.
4. Dispatch to payload exporter without running normal bundle export.
```

Acceptance:

```text
Invalid payload export invocations fail before mounting paks.
Normal export, verify-only, context-only, and semantic-only behavior is unchanged.
```

### T2.2 Implement Referenced Texture Collection

Status: done

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/BundleTexturePayloadExporter.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/Semantic/MaterialLayerContractExporter.cs
```

Input priority:

```text
1. analysis/material_layer_parameter_bindings.json
2. parameters/textures.json
3. analysis/texture_assets.json
4. source/textures/*.cooked.json metadata hints only
```

Tasks:

```text
1. Gather unique textures by normalized ObjectPath + TextureName.
2. Preserve StableKey and Unity property references.
3. Attach ImportIntent and ColorSpace using existing heuristics.
4. Report candidates missing ObjectPath instead of guessing.
```

Acceptance:

```text
referenced_textures scope includes every material texture reference with a resolvable ObjectPath.
Duplicate parameters referencing the same texture produce one payload file with multiple property references.
```

### T2.3 Reuse Texture Decode And Sidecar Writing

Status: done

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/MissingUnityTextureExporter.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/BundleTexturePayloadExporter.cs
```

Tasks:

```text
1. Extract shared decode/export helpers from MissingUnityTextureExporter if needed.
2. Decode UTexture2D, UTexture2DArray first slice, and UTextureCube panorama consistently with existing exporter.
3. Write PNG output first.
4. Preserve alpha channel when available.
5. Write sidecar JSON next to each image.
```

Acceptance:

```text
Payload export and ExportMissingTextures produce compatible sidecar metadata.
Existing MissingUnityTextureExporter behavior is not regressed.
```

### T2.4 Write Payload Manifest

Status: done

File:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/BundleTexturePayloadExporter.cs
```

Tasks:

```text
1. Write texture_payload/manifest.json.
2. Include all exported, skipped, failed, and missing candidate statuses.
3. Compute Sha256 for exported payload files.
4. Store bundle-relative paths only.
5. Record warnings for virtual textures, decode failures, or ambiguous metadata.
```

Acceptance:

```text
Manifest is parseable JSON and can drive downstream import without original game data.
All listed payload paths exist unless item status is failed or missing.
```

## Phase 3: Payload Import Helper

### T3.1 Implement Import Helper

Status: done

New file:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Tools/import_bundle_texture_payload.py
```

Arguments:

```text
--bundle <Bundle>
--restore-report <DryRunReport.json>
--unity-assets-root <UnityProject>/Assets
--texture-out Assets/Art/Recovered/Subnautica2
--overwrite-textures
--report-out <Bundle>/unity_texture_payload_import_report.json
```

Tasks:

```text
1. Read texture_payload/manifest.json.
2. Read MissingTextureExportCandidates from restore report.
3. Match candidates to payload items by ObjectPath and TextureName.
4. Copy only needed payload images and sidecars into Unity Assets.
5. Skip existing Unity files unless --overwrite-textures is explicit.
6. Never modify .mat or .meta files.
```

Acceptance:

```text
Downstream workspace can recover missing texture files from bundle payload without original game paks.
Import report lists copied, skipped, and missing payload entries.
```

### T3.2 Add Import Report

Status: done

Output:

```text
<Bundle>/unity_texture_payload_import_report.json
```

Tasks:

```text
1. Include source payload manifest path.
2. Include UnityAssetsRoot and TextureOut.
3. Include copied/skipped/missing/failed items.
4. Include MatFilesModified = false.
5. Include next step: let Unity import, then rerun DryRun.
```

Acceptance:

```text
Report is parseable and enough for a fresh Agent to continue.
No local absolute paths are written into bundle-local persistent reports unless they are explicitly marked runtime diagnostics.
```

## Phase 4: Verifier

### T4.1 Validate Payload When Present

Status: done

File:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/BundleVerifier.cs
```

Tasks:

```text
1. Do not require texture_payload by default.
2. If texture_payload/manifest.json exists, parse it.
3. Validate schema.
4. Validate payload paths are relative and stay inside texture_payload.
5. Validate listed files exist for exported/skipped entries.
6. Validate sidecar paths exist when declared.
7. Validate Sha256 when present.
8. Fail on machine-local absolute paths in manifest or sidecars.
```

Acceptance:

```text
--verify-only remains OK for bundles without payload.
--verify-only catches broken payload manifests when payload exists.
```

### T4.2 Update Agent Context

Status: done

File:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/AgentWorkspaceWriter.cs
```

Tasks:

```text
1. Add TexturePayload.Exists.
2. Add TexturePayload.Manifest when present.
3. Add TexturePayload.Mode and item count if readable.
4. Add ReadFirst entry for texture_payload/manifest.json only when present.
5. Add constraint that payload import belongs to material restore only.
```

Acceptance:

```text
agent_context.json gives fresh Agents a machine-readable way to discover payload availability.
agent_context.json remains path-portable.
```

## Phase 5: Workflow Validation

### T5.1 Export Payload Fixture

Status: done

Fixture:

```text
Any known material bundle with texture parameters and original UE cooked data available.
```

Tasks:

```text
1. Run full bundle export.
2. Run --export-bundle-texture-payload --payload-scope referenced_textures.
3. Run --verify-only.
4. Check texture_payload/manifest.json and image files.
```

Acceptance:

```text
Payload export succeeds or reports actionable per-texture failures.
Verify succeeds when payload is complete and portable.
```

### T5.2 Consume Payload Without Game Data

Status: done

Tasks:

```text
1. Copy bundle to a clean workspace that does not have game paks configured.
2. Run material restore DryRun against a Unity project missing some textures.
3. Run import_bundle_texture_payload.py.
4. Let Unity import textures and generate .meta files.
5. Rerun DryRun.
```

Acceptance:

```text
MissingTextureGuids decreases or reaches zero without original game data.
No .mat changes happen before explicit Apply.
```

### T5.3 Preserve Old ExportMissingTextures Flow

Status: done

Tasks:

```text
1. Run existing ExportMissingTextures on a bundle without payload.
2. Confirm it still reads MissingTextureExportCandidates.
3. Confirm it still decodes from original game data.
4. Confirm reports/sidecars are unchanged or backward-compatible.
```

Acceptance:

```text
Existing local recovery workflow continues to work.
```

## Phase 6: Batch Distribution

### T6.1 Add Batch Include Option

Status: done

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/ShaderBundleExportOptions.cs
Batch export implementation files
```

Tasks:

```text
1. Add --include-texture-payload to batch export.
2. For first version, generate embedded per-bundle payloads.
3. Do not deduplicate across bundles yet.
4. Record payload status in batch root agent_context.json.
```

Acceptance:

```text
Batch-exported bundles can be distributed independently.
```

## Overall Acceptance

```text
1. Bundle docs explain that default bundles do not include raw/decoded texture image resources.
2. Users have two first-step Goal choices: lightweight bundle export and bundle export with texture payload.
3. Explicit payload export creates a portable texture_payload directory.
4. Payload import can recover missing Unity textures without original cooked game data.
5. Existing ExportMissingTextures still works when original cooked game data is available.
6. Shader reconstruction remains free of texture export/import side effects.
7. Material Apply remains guarded by the standalone Apply token and --apply-confirm WRITE_MAT.
8. Fresh Agents get clear blocker messaging when neither texture_payload nor original game data is available.
```

## Implementation Verification

Verified on 2026-06-12.

Build:

```text
dotnet build CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release --no-restore
  Build succeeded, 0 warnings, 0 errors.
  Local note: cmake is not installed, so CUE4Parse-Natives reports build failed and continues without native binaries.
```

Static checks:

```text
python -m py_compile CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\import_bundle_texture_payload.py
  OK

dotnet run ... -- --export-bundle-texture-payload --bundle <missing bundle>
  Fails before provider mount with missing --game, --paks, --mapping, and missing bundle errors.
```

Payload verifier fixture:

```text
Created a temporary bundle-local texture_payload fixture under a local temp test bundle.
dotnet run ... -- --verify-only <fixture bundle> --verbose
  Verify: OK
```

Real payload export fixture:

```text
Source bundle:
  <TempPayloadExportTestBundle>

Command:
  dotnet run ... -- --export-bundle-texture-payload --game Subnautica2 --paks <GamePaks> --mapping <MappingUsmap> --bundle <bundle> --payload-scope referenced_textures

Result:
  Exported: 15
  Skipped existing: 0
  Failed: 0
  Missing ObjectPath: 0
  texture_payload/manifest.json Schema: ue-bundle-texture-payload/v1
  PayloadMode: referenced_textures
  Verify: OK
  agent_context.json TexturePayload.Exists: true
  agent_context.json ReadFirst includes texture_payload/manifest.json: true
```

Payload import fixture:

```text
python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\import_bundle_texture_payload.py --bundle <bundle> --restore-report <DryRun fixture> --unity-assets-root <TemporaryUnityAssets> --texture-out Assets/Art/Recovered/Subnautica2

Result:
  Copied: 1
  Skipped existing: 0
  Missing payload: 0
  Failed: 0
  Report: unity_texture_payload_import_report.json
  MatFilesModified: false
```

Existing ExportMissingTextures regression:

```text
dotnet run ... -- --export-missing-unity-textures --game Subnautica2 --paks <GamePaks> --mapping <MappingUsmap> --bundle <bundle> --restore-report <DryRun fixture> --unity-assets-root <TemporaryUnityAssets> --texture-out Assets/Art/Recovered/Subnautica2

Result:
  Exported: 1
  Skipped existing: 0
  Failed: 0
```

