# UE Multi-Bundle Workspace Reconstruction Tasks

Status: proposed task breakdown for `Doc/UE_Multi_Bundle_Workspace_Reconstruction_Plan.md`.

This task list implements the multi-bundle material reconstruction workspace while preserving the existing single-bundle handoff workflow.

## Task Status Legend

- `TODO`: not started.
- `PARTIAL`: some supporting behavior exists, but the task is not complete.
- `DONE`: implemented and verified.
- `BLOCKED`: cannot proceed without external input.

## P0 - Contracts And Documentation

### MBW-T001 - Define Single-Bundle vs Multi-Bundle Workspace Rules

Status: DONE

Goal:

Document the mode boundary so future Agents do not confuse single-bundle convenience launchers with multi-bundle batch workflows.

Files:

```text
AGENTS.md
Doc/UE_Cooked_Material_Bundle_Export_Goal.md
Doc/UE_Cooked_Material_Bundle_With_TexturePayload_Export_Goal.md
Doc/UE_Multi_Bundle_Workspace_Reconstruction_Plan.md
```

Implementation:

1. State that bare `/goal <GoalFile>.md` is valid only when the current workspace contains exactly one `*.bundle`.
2. State that multi-bundle workspaces require explicit `Bundle=...`, explicit `Bundle\Goal.md`, or a workspace-level batch goal.
3. State that Agents must stop and ask for `Bundle=...` when zero or multiple bundles are present.
4. State that single-bundle self-contained exports remain supported.

Acceptance:

- A new Agent reading the docs can tell which mode it is in before running any goal.
- No canonical workflow tells users to run ambiguous bare bundle-local goals in a multi-bundle workspace.

### MBW-T002 - Update Human Workflow Docs

Status: DONE

Goal:

Update human-facing workflow docs to show both single-bundle and multi-bundle usage.

Files:

```text
K:/WP_ShaderReconstruction/Doc/Workflow.md
K:/WP_ShaderReconstruction/Doc/Workflow_Simple.md
K:/WP_ShaderReconstruction/Doc/Workflow_OneClick_Goal.md
```

Implementation:

1. Add a short "single bundle" section.
2. Add a short "multi bundle" section.
3. Explain that multi-bundle workspaces should use:

```text
/goal MI_Target.bundle\UE_Unity_Material_OneClick_NoVisual_Reconstruction_Goal.md -mat ...
```

or:

```text
/goal UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md MatMap=MaterialMap.json
```

4. Explain that new RenderDoc data should be placed under `Bundle/RenderDocCapture`.
5. Explain that shared textures live under workspace `Textures`.

Acceptance:

- The docs no longer imply that bare local goal filenames are generally valid in all workspaces.
- A person can follow the docs for both one material and many materials.

### MBW-T003 - Add Multi-Bundle Workspace Goal Doc

Status: DONE

Goal:

Add a canonical workspace-level batch NoVisual goal.

Files:

```text
Doc/UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md
```

Implementation:

1. Define inputs:

```text
MatMap=MaterialMap.json
UnityRoot=<UnityProject>
Filter=<optional glob>
ContinueOnError=true|false
```

2. Define `MaterialMap.json` schema.
3. Define serial Unity write policy.
4. Define per-bundle output report requirements.
5. Define success/failure behavior.
6. Explicitly forbid visual validation, screenshots, semantic capture, and RenderDoc processing in this NoVisual batch goal.

Acceptance:

- A new Agent can run the batch NoVisual workflow from only this goal document plus a material map.
- The goal cannot be interpreted as permission to run visual validation.

## P1 - Exporter Workspace Layout

### MBW-T010 - Keep Bare Goal Launchers Single-Bundle Only

Status: DONE

Goal:

Ensure workspace-root goal launchers are generated only when the bundle parent contains exactly one `*.bundle`.

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/AgentWorkspaceWriter.cs
```

Implementation:

1. Count `*.bundle` directories in the bundle parent.
2. Generate root goal launchers only when count is exactly one.
3. Do not generate launchers for batch roots or directories with multiple bundles.
4. In generated bundle docs, state that bare launchers are single-bundle convenience files only.

Acceptance:

- Single-bundle export produces root launchers.
- Two bundles in one workspace do not produce ambiguous root bundle-local launchers.
- `--context-only` follows the same rule.

### MBW-T011 - Add Multi-Bundle Batch Root Files

Status: DONE

Goal:

When exporting multiple bundles into one workspace, generate workspace-level batch handoff docs.

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/AgentWorkspaceWriter.cs
Doc/UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md
```

Generated files:

```text
Workspace/
  AGENTS.md
  WORKFLOW.md
  NEXT_TASK.md
  PROMPT_NEXT_SESSION.md
  UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md
  agent_context.json
  batch_manifest.json
```

Implementation:

1. Detect batch/multi-bundle export mode.
2. Write batch-root docs that describe workspace-level behavior.
3. Include the bundle list, texture mode, and material paths in `batch_manifest.json`.
4. Do not generate ambiguous bundle-local root launchers in this mode.

Acceptance:

- A multi-bundle workspace has exactly one batch entrypoint.
- Batch docs point users to explicit bundle paths for per-material work.

### MBW-T012 - Update Verifier For Workspace Modes

Status: DONE

Goal:

Make verification aware of single-bundle and multi-bundle rules.

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/BundleVerifier.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/AgentWorkspaceWriter.cs
```

Implementation:

1. Single-bundle verify keeps checking required bundle-local files.
2. Add optional workspace-root launcher checks only when parent has exactly one bundle.
3. Add batch-root verify for `batch_manifest.json` and workspace batch docs.
4. Verify `agent_context.json` and `batch_manifest.json` are parseable.

Acceptance:

- `--verify-only Bundle` still works for a single bundle.
- Batch-root verification catches missing batch docs.
- Verifier does not fail a multi-bundle workspace because root bundle-local launchers are absent.

## P2 - Shared Texture Payload Store

### MBW-T020 - Add Texture Payload Mode Option

Status: DONE

Goal:

Support explicit texture payload modes.

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Options/ShaderBundleExportOptions.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Program.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/*
```

CLI:

```text
--texture-payload-mode shared
--texture-payload-mode bundle
--texture-payload-mode none
--texture-root <path>
```

Implementation:

1. Add enum/string option for payload mode.
2. Default to `shared` for multi-bundle/workspace exports.
3. Preserve `bundle` mode for self-contained export.
4. Preserve `none` mode for lightweight bundle export.
5. Add validation for invalid combinations.

Acceptance:

- Exporter accepts all three modes.
- Invalid mode fails with a clear error.
- Existing lightweight exports remain possible.

### MBW-T021 - Implement Shared Textures Manifest

Status: DONE

Goal:

Write deduplicated texture payloads to workspace `Textures`.

Files:

```text
Workspace/Textures/manifest.json
Workspace/Textures/payload/...
Bundle/parameters/textures.json
Bundle/texture_payload_manifest.json
```

Implementation:

1. Create `<bundleParent>/Textures` when mode is `shared`.
2. Compute dedup key:

```text
UE ObjectPath + cooked bulk hash + pixel format + exported mip policy
```

3. Write payload file only when the dedup key is missing.
4. Record payload entries in shared `Textures/manifest.json`.
5. Record per-bundle references using relative paths to `../Textures`.

Acceptance:

- Two materials using the same texture write one shared payload copy.
- Bundles can still identify every texture they need.
- No dedup is performed by filename alone.

### MBW-T022 - Preserve Bundle-Local Texture Payload Mode

Status: DONE

Goal:

Keep self-contained texture export behavior for single-bundle distribution.

Files:

```text
Bundle/texture_payload/manifest.json
Bundle/texture_payload/payload/...
Bundle/agent_context.json
Bundle/manifest.json
```

Implementation:

1. When mode is `bundle`, write textures inside the bundle.
2. Mark the bundle as self-contained in `agent_context.json`.
3. Ensure material restore can resolve bundle-local texture payloads without the original game data.

Acceptance:

- A bundle exported with `--texture-payload-mode bundle` can be copied alone and still restore textures.

### MBW-T023 - Update Material Restore Texture Lookup

Status: DONE

Goal:

Material restore should resolve textures from both bundle-local and shared payload stores.

Files:

```text
tools/shader_reconstruction/unity_material_apply_ue_params.py
Doc/UE_Unity_Material_Property_Restore_Goal.md
```

Lookup order:

1. Bundle-local texture payload.
2. Shared workspace `Textures`.
3. Existing Unity project assets.
4. Explicit missing texture export from original game data.

Implementation:

1. Parse payload mode from bundle manifests.
2. Load shared `Textures/manifest.json` if present.
3. Use relative paths from bundle to shared store.
4. Keep DryRun behavior read-only.
5. Export missing textures into the correct store based on payload mode.

Acceptance:

- DryRun reports whether textures are resolved from bundle, shared store, Unity project, or missing.
- `ExportMissingTextures` writes to shared store when mode is `shared`.
- `ExportMissingTextures` writes to bundle payload when mode is `bundle`.

## P3 - RenderDoc Bundle-Local Capture Layout

### MBW-T030 - Prefer Bundle/RenderDocCapture

Status: DONE

Goal:

Move the default RenderDoc capture location under the matching bundle.

Files:

```text
Doc/UE_RenderDoc_Compact_Summary_Goal.md
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/AgentWorkspaceWriter.cs
```

Layout:

```text
Bundle/
  RenderDocCapture/
    EID_...
  analysis/renderdoc/
```

Implementation:

1. Update generated docs to instruct users to place captures under `Bundle/RenderDocCapture`.
2. Update canonical goal docs.
3. Keep legacy workspace-root `RenderDocCapture` detection only as a fallback.
4. If multiple bundles exist and capture is outside a bundle, require `Bundle=...`.

Acceptance:

- New docs no longer recommend workspace-root RenderDocCapture as the default.
- RenderDoc compact summary reads `Bundle/RenderDocCapture` without extra parameters.

### MBW-T031 - Update RenderDoc Compact Summary Auto-Detection

Status: DONE

Goal:

Make compact summary tooling correctly bind capture data to a bundle in multi-bundle workspaces.

Files:

```text
tools/shader_reconstruction/renderdoc_compact_summary/*
Doc/UE_RenderDoc_Compact_Summary_Goal.md
```

Implementation:

1. If invoked from inside a bundle, use `./RenderDocCapture`.
2. If invoked from workspace root with explicit `Bundle=...`, use `Bundle/RenderDocCapture`.
3. If old `Workspace/RenderDocCapture` exists and exactly one bundle exists, allow it as legacy input.
4. If multiple bundles exist and capture is not under a bundle, stop and ask.

Acceptance:

- Multi-bundle workspace capture association is deterministic.
- No RenderDoc summary is written into the wrong bundle.

## P4 - Batch NoVisual Orchestrator

### MBW-T040 - Define Generated MaterialMap Schema

Status: DONE

Goal:

Define a stable generated mapping from bundle to Unity `.mat`. The preferred source is `UnityMatDir=...`, not a hand-authored map.

Files:

```text
Doc/UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md
Workspace/MaterialMap.json
Workspace/MaterialMap.example.json
```

Schema:

```json
{
  "schema": "ue-unity-material-map/v1",
  "generatedFrom": "unity_mat_dir",
  "unityRoot": "<UnityProjectRoot>",
  "items": [
    {
      "bundle": "MI_A.bundle",
      "mat": "Assets/.../MI_A.mat",
      "materialName": "MI_A"
    }
  ]
}
```

Implementation:

1. Exporter scans `UnityMatDir` recursively for `.mat`.
2. Convert each `Assets/.../*.mat` path to the exact UE `/Game/...` material path.
3. Generate `MaterialMap.json` automatically.
4. Store `mat` as an `Assets/...` relative path when a Unity project root can be inferred.
5. Allow `UnityRoot=...` override when the workspace is moved to another machine.
6. Keep `MaterialMap.example.json` as fallback documentation for manual repair only.
7. Require explicit bundle path per item.
8. Require explicit `.mat` path per item.
9. Store `unrealMaterialPath` per item.
10. Allow duplicate `.mat` file names when their `Assets/...` paths differ.
11. Use path-derived bundle names when duplicate file names would collide.
12. Treat a missing path-derived `/Game/...` material as not found even if a same-name material exists elsewhere.
13. Allow optional tags/filter fields.
14. Validate duplicate bundle entries.
15. Validate duplicate mat entries and report warnings.

Acceptance:

- `UnityMatDir` batch export writes `MaterialMap.json`.
- Duplicate Unity `.mat` file names in different folders resolve through their `Assets/... -> /Game/...` paths.
- Batch NoVisual can run from the workspace with no `MatMap=...` argument.
- Invalid map fails before any Unity writes.
- A valid generated or manually supplied map can drive the batch orchestrator.

### MBW-T041 - Implement Batch NoVisual Runner

Status: DONE

Goal:

Run NoVisual reconstruction over multiple bundles with serialized Unity writes.

Candidate file:

```text
tools/shader_reconstruction/workspace_batch_no_visual.py
```

Implementation:

1. Load `MaterialMap.json` by default, or a user supplied `--mat-map`.
2. Resolve every bundle.
3. Verify every bundle before processing.
4. For each item:
   - run shader reconstruction step;
   - run material restore DryRun;
   - export missing textures if explicitly requested by batch options;
   - run material restore Apply only with explicit apply confirmation;
   - run shader compile/import check;
   - write per-bundle handoff reports.
5. Serialize all Unity project writes.
6. Support `ContinueOnError`.

Acceptance:

- Two mapped bundles can be processed sequentially.
- Failed item does not corrupt later items.
- Batch report records command, exit code, report path, and status per item.

### MBW-T042 - Add Workspace Batch Reports

Status: DONE

Goal:

Write a compact batch summary for humans and future Agents.

Output:

```text
Workspace/batch_reports/no_visual_batch_report.md
Workspace/batch_reports/no_visual_batch_report.json
```

Implementation:

1. Record each bundle item status:
   - success;
   - failed;
   - skipped;
   - needs_input.
2. Record shader path, mat path, compile status, restore status.
3. Record handoff summary path.
4. Record texture store mode.
5. Avoid embedding large logs; store paths and byte counts.

Acceptance:

- A user can see which materials are ready for visual validation.
- A future Agent can resume failed items from the report.

### MBW-T043 - Enforce NoVisual Scope

Status: DONE

Goal:

Prevent batch NoVisual from drifting into visual validation or RenderDoc processing.

Files:

```text
Doc/UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md
tools/shader_reconstruction/workspace_batch_no_visual.py
```

Implementation:

1. Do not call semantic visual validation.
2. Do not call lightweight smoke validation.
3. Do not capture screenshots.
4. Do not run RenderDoc compact summary.
5. Do require shader compile/import success.

Acceptance:

- A batch item cannot be marked success if Unity shader import/compile has errors.
- A batch item can be marked success without any RenderDoc data or visual references.

## P5 - Per-Bundle NoVisual Handoff

### MBW-T050 - Generate NoVisual Summary JSON

Status: DONE

Goal:

Write machine-readable reconstruction summary into each bundle.

Output:

```text
Bundle/analysis/unity_no_visual_reconstruction_summary.json
```

Fields:

```text
material_name
ue_material_path
bundle_path
unity_mat_path
unity_shader_path
shader_reuse_key
shader_reuse_decision
compiled
compile_errors
restored_parameter_count
missing_textures
texture_payload_mode
implemented_modules
approximate_features
omitted_features
next_recommended_goal
```

Acceptance:

- The JSON is parseable.
- It contains enough information to resume visual validation without chat history.

### MBW-T051 - Generate NoVisual Handoff Markdown

Status: DONE

Goal:

Write human-readable handoff notes for the next visual validation session.

Output:

```text
Bundle/analysis/unity_no_visual_reconstruction_summary.md
Bundle/analysis/unity_handoff_for_visual_validation.md
```

Implementation:

1. Summarize what was generated or reused.
2. List restored parameters and unresolved parameters.
3. List missing/unresolved textures.
4. List implemented Master/Layer/Blend modules.
5. List approximate or omitted runtime features.
6. State whether RenderDoc data exists.
7. State where to place RenderDoc data:

```text
Bundle/RenderDocCapture
```

8. Provide recommended next command:

```text
/goal Bundle\UE_Unity_Material_Semantic_Visual_Validation_Goal.md Mat=<UnityMat>
```

or, in a single-bundle workspace:

```text
/goal UE_Unity_Material_Semantic_Visual_Validation_Goal.md Mat=<UnityMat>
```

Acceptance:

- A different person can start visual validation from this file.
- The file clearly states whether the bundle is self-contained or depends on shared `Textures`.

### MBW-T052 - Mirror Handoff Paths Into Agent Context

Status: DONE

Goal:

Make handoff reports discoverable by future Agents.

Files:

```text
Bundle/agent_context.json
Bundle/analysis/unity_no_visual_reconstruction_summary.json
Bundle/analysis/unity_handoff_for_visual_validation.md
```

Implementation:

1. Add `UnityReconstruction.NoVisualSummary`.
2. Add `UnityReconstruction.VisualValidationHandoff`.
3. Add `TexturePayload.Mode`.
4. Add `TexturePayload.SharedRoot` when applicable.

Acceptance:

- A future Agent can find the handoff files by reading `agent_context.json`.

## P6 - Testing And Regression

### MBW-T060 - Single-Bundle Regression Test

Status: DONE

Goal:

Ensure the existing one-material flow still works.

Steps:

1. Export one bundle into an empty workspace.
2. Use `--texture-payload-mode bundle`.
3. Verify root launchers exist.
4. Run `--verify-only`.
5. Confirm material restore can find bundle-local textures.

Acceptance:

- Existing single-bundle user flow is not broken.

### MBW-T061 - Multi-Bundle Workspace Export Test

Status: DONE

Goal:

Verify multi-bundle workspace behavior.

Steps:

1. Export two material bundles into the same workspace.
2. Use `--texture-payload-mode shared`.
3. Confirm no ambiguous bundle-local root launchers exist.
4. Confirm batch-root docs and `batch_manifest.json` exist.
5. Run verifier.

Acceptance:

- Multi-bundle workspace is deterministic and unambiguous.

### MBW-T062 - Shared Texture Dedup Test

Status: DONE

Goal:

Verify shared texture payload dedup.

Steps:

1. Pick two materials that share at least one texture.
2. Export both with shared texture payload.
3. Inspect `Textures/manifest.json`.
4. Confirm the shared texture payload appears once.
5. Confirm both bundle manifests point to it.

Acceptance:

- Shared texture storage saves disk space without breaking per-bundle references.

### MBW-T063 - Batch NoVisual Test

Status: DONE

Goal:

Verify workspace batch NoVisual execution.

Steps:

1. Create `MaterialMap.json` for two bundles.
2. Run:

```text
/goal UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md MatMap=MaterialMap.json
```

3. Confirm Unity writes are serialized.
4. Confirm each bundle has NoVisual handoff outputs.
5. Confirm workspace batch report is written.

Acceptance:

- Batch completes with per-material status.
- No item is marked success with shader compile/import errors.

### MBW-T064 - RenderDoc Bundle-Local Detection Test

Status: DONE

Goal:

Verify RenderDoc compact summary uses bundle-local capture folders.

Steps:

1. Put drawcall export data under:

```text
MI_A.bundle/RenderDocCapture
```

2. Run RenderDoc compact summary for `MI_A.bundle`.
3. Confirm output goes to:

```text
MI_A.bundle/analysis/renderdoc
```

4. Add `MI_B.bundle` and ensure no capture ambiguity occurs.

Acceptance:

- RenderDoc evidence cannot be accidentally attached to the wrong bundle.

### MBW-T065 - Fresh Agent Handoff Test

Status: DONE

Goal:

Verify a new session can continue visual validation from a NoVisual handoff.

Steps:

1. Open only one completed bundle and its handoff summary.
2. Do not use prior chat history.
3. Confirm the Agent can identify:
   - Unity mat path;
   - Unity shader path;
   - missing textures;
   - implemented modules;
   - approximate features;
   - next visual validation goal.

Acceptance:

- Handoff summary is sufficient for follow-up visual validation and effect alignment.

## Recommended Implementation Order

1. MBW-T001, MBW-T002, MBW-T003
2. MBW-T010, MBW-T011, MBW-T012
3. MBW-T020, MBW-T021, MBW-T022, MBW-T023
4. MBW-T030, MBW-T031
5. MBW-T040, MBW-T041, MBW-T042, MBW-T043
6. MBW-T050, MBW-T051, MBW-T052
7. MBW-T060 through MBW-T065

## Completion Criteria

The multi-bundle workspace workflow is complete when:

1. Single-bundle exports still work with bare local goal launchers.
2. Multi-bundle workspaces have no ambiguous bare bundle-local goal launchers.
3. Shared workspace textures deduplicate payloads and restore correctly.
4. RenderDoc captures are associated with the correct bundle.
5. Batch NoVisual reconstruction can process multiple bundles serially.
6. Each processed bundle gets a complete handoff summary.
7. Fresh Agents can continue visual validation from a bundle without prior chat history.

## Implementation Validation Notes

Validated on 2026-06-15:

- `dotnet build CUE4Parse.ShaderBundleExporter -c Release --no-restore` succeeds; the local optional native build still reports missing `cmake` and continues.
- Single-bundle regression: exported `MI_CG_CoralDomeBroken_01a` to `D:/Tmp/mbw_real_single/MI_CG_CoralDomeBroken_01a.bundle` with `--include-texture-payload --texture-payload-mode bundle`; bundle `--verify-only` passed and root single-bundle launchers were generated.
- Multi-bundle regression: exported `MI_CG_CoralDomeBroken_01a` and `MI_CG_RockSmooth_01a` to `D:/Tmp/mbw_real_multi3` with `--include-texture-payload --texture-payload-mode shared`; batch root `--verify-only` passed and root one-click launchers were absent.
- Shared texture dedup: `D:/Tmp/mbw_real_multi3/Textures/manifest.json` contains 20 unique items; shared `T_RockLichenMasks_01a` and `Curve_Base_Atlas` each appear once across the two bundles.
- Generated `commands/verify_all_bundles.ps1` passed on the real multi-bundle workspace.
- Batch NoVisual helper wrote per-bundle handoff summaries and workspace reports; without Unity compile reports it returned non-zero and marked both items `needs_input`, not `success`.
- Bundle context refresh after NoVisual handoff updates `agent_context.json.UnityReconstruction` and bundle/root verification still passes.
- RenderDoc compact summary detection was tested with bundle-local positive input and multi-bundle root-level ambiguity; ambiguous root-level `RenderDocCapture` is rejected.
