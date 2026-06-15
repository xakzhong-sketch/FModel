# UE Multi-Bundle Workspace Reconstruction Plan

Status: proposed

This plan upgrades the cooked UE material reconstruction workflow from a single-bundle handoff model to a multi-bundle workspace model while preserving the existing self-contained single-bundle path.

## Goals

1. Support a material reverse-engineering workspace that contains multiple `*.bundle` directories.
2. Keep single-bundle exports self-contained and easy to distribute.
3. Move new RenderDoc drawcall exports under the matching bundle to remove capture-to-material ambiguity.
4. Add a shared workspace texture payload store to deduplicate texture exports across many materials.
5. Add a batch NoVisual reconstruction workflow for running shader reconstruction, material restore, and compile/import checks over many bundles.
6. Generate a per-bundle handoff summary after NoVisual reconstruction so another Agent/person can later continue visual validation and effect alignment without reading chat history.

## Non-Goals

- Do not remove the current single-bundle workflow.
- Do not require RenderDoc evidence for NoVisual batch reconstruction.
- Do not run visual validation in the NoVisual batch workflow.
- Do not parallel-write the same Unity project from multiple Agents.
- Do not make workspace-root bare `/goal <file>.md` ambiguous in multi-bundle workspaces.

## Workspace Modes

### Single-Bundle Workspace

Use this mode when one material is exported and handed to another person independently.

Expected layout:

```text
Workspace/
  MI_Target.bundle/
  PROMPT_NEXT_SESSION.md
  UE_RenderDoc_Compact_Summary_Goal.md
  UE_Unity_Material_OneClick_Reconstruction_Goal.md
  UE_Unity_Material_OneClick_NoVisual_Reconstruction_Goal.md
  UE_Unity_Material_Semantic_Visual_Validation_Goal.md
  UE_Unity_Material_Lightweight_Visual_Smoke_Goal.md
```

Behavior:

- Workspace-root bare goal launchers are allowed only when the workspace contains exactly one `*.bundle`.
- `/goal UE_Unity_Material_OneClick_NoVisual_Reconstruction_Goal.md -mat ...` resolves to the only bundle-local goal.
- Texture payload may be stored inside the bundle for self-contained distribution.

### Multi-Bundle Workspace

Use this mode when one workspace contains many material bundles.

Expected layout:

```text
Workspace/
  Textures/
    manifest.json
    payload/
      Game/...
  MI_A.bundle/
  MI_B.bundle/
  MI_C.bundle/
  UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md
  batch_manifest.json
```

Behavior:

- Workspace-root bare goal launchers must not be generated for bundle-local goals when more than one `*.bundle` exists.
- Agents must use explicit bundle paths for single-bundle operations:

```text
/goal MI_A.bundle\UE_Unity_Material_OneClick_NoVisual_Reconstruction_Goal.md -mat <UnityProject>\Assets\...\MI_A.mat
```

- Batch operations should use a workspace-level batch goal:

```text
/goal UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md MatMap=MaterialMap.json
```

- If an Agent receives a bare bundle-local goal in a multi-bundle workspace, it must stop and ask for `Bundle=...`; it must not guess.

## RenderDoc Layout Change

New preferred layout:

```text
MI_Target.bundle/
  RenderDocCapture/
    EID_...
  analysis/
    renderdoc/
```

Rules:

- New docs and goals should instruct users to put drawcall exports in `Bundle/RenderDocCapture`.
- RenderDoc compact summary auto-detection should prefer `Bundle/RenderDocCapture`.
- For backward compatibility, the analyzer may still detect old workspace-root `RenderDocCapture`, but it must require `Bundle=...` if multiple bundles exist.
- Compact RenderDoc output remains under `Bundle/analysis/renderdoc`.

## Texture Payload Store

### Modes

Add an explicit export option:

```text
--texture-payload-mode shared
--texture-payload-mode bundle
--texture-payload-mode none
```

Default:

- `shared` for new multi-bundle exports and workspace goals.
- `bundle` remains available for single material distribution.
- `none` keeps current lightweight bundle behavior without texture payload.

### Shared Store Layout

```text
Workspace/
  Textures/
    manifest.json
    payload/
      Game/Art/Environment/...
```

Each bundle keeps a texture reference manifest, but the payload path points to the shared store:

```json
{
  "texturePayloadMode": "shared",
  "workspaceTextureRoot": "../Textures",
  "textures": [
    {
      "ueObjectPath": "/Game/...",
      "sourceHash": "...",
      "format": "BC7",
      "relativePayloadPath": "../Textures/payload/Game/..."
    }
  ]
}
```

Dedup key:

```text
UE ObjectPath + cooked bulk hash + pixel format + exported mip policy
```

Do not deduplicate by filename alone.

### Bundle Mode Layout

```text
MI_Target.bundle/
  texture_payload/
    manifest.json
    payload/
```

Bundle mode stays fully self-contained and is preferred when a bundle will be sent to someone who does not have the original game data or shared workspace textures.

## Batch NoVisual Workflow

Add a workspace-level goal:

```text
UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md
```

Inputs:

```text
MatMap=MaterialMap.json optional; defaults to workspace MaterialMap.json
UnityRoot=<UnityProject>
Filter=<optional bundle/material glob>
ContinueOnError=true|false
```

Preferred batch export path:

```text
1. User provides UnityMatDir=<UnityProject>/Assets/... or UnityScene=<UnityProject>/Assets/.../Scene.unity.
2. For UnityMatDir, exporter recursively scans *.mat files.
   For UnityScene, exporter scans the scene and referenced Unity text assets/prefabs for material GUIDs and resolves them to .mat files.
3. Each Assets-relative .mat path is mapped to the matching UE /Game material path.
4. Exporter writes one bundle per material plus workspace MaterialMap.json.
5. Batch NoVisual goal can run without a MatMap argument.
```

Path mapping example:

```text
Assets/Art/Environment/Biome/CoralGarden/Rocks/Material/MI_CG_RockPebbles_01a.mat
  ->
/Game/Art/Environment/Biome/CoralGarden/Rocks/Material/MI_CG_RockPebbles_01a
```

Duplicate `.mat` file names are allowed when their Assets-relative paths differ. The exporter should use the path-derived /Game material path for cooked data resolution and a path-derived bundle directory name when needed to avoid collisions.

If the path-derived `/Game/...` material does not exist, the item is not found. A same-name material under another UE folder must not be substituted.

UnityScene mode is a filtered batch export. It should not export every material under a directory; it should export only the `.mat` assets found from the scene dependency scan.

Generated `MaterialMap.json` example:

```json
{
  "unityRoot": "<UnityProjectRoot>",
  "items": [
    {
      "bundle": "MI_A.bundle",
      "mat": "Assets/.../MI_A.mat",
      "materialName": "MI_A",
      "unrealMaterialPath": "/Game/.../MI_A"
    },
    {
      "bundle": "MI_B.bundle",
      "mat": "Assets/.../MI_B.mat",
      "materialName": "MI_B"
    }
  ]
}
```

If the workspace is moved to another machine, pass `UnityRoot=<UnityProjectRoot>` to rebind the generated `Assets/...` paths.

Batch steps per item:

1. Verify the bundle.
2. Run bundle-local NoVisual one-click reconstruction.
3. Restore material properties with explicit DryRun, missing texture handling, and Apply only when required confirmation is present in the substep.
4. Validate Unity shader import/compile status.
5. Do not run visual validation, semantic capture, smoke validation, RenderDoc summary, or screenshot comparison.
6. Write per-bundle NoVisual handoff reports.
7. Continue or stop based on `ContinueOnError`.

## Parallelism Policy

Recommended implementation:

- The batch orchestrator runs bundle items sequentially when writing to the same Unity project.
- Read-only evidence summarization may be parallelized internally, but Unity project writes, `.mat` edits, shader imports, and compile checks must be serialized.
- Do not spawn one write-capable SubAgent per material against the same Unity project.
- Parallel NoVisual reconstruction is allowed only when each worker has an isolated Unity project copy or a separate branch/worktree with no shared `Assets` writes.

Reason:

- Unity `AssetDatabase`, `.meta` files, shader import, material serialization, and script refresh are not safe under uncontrolled concurrent writes.

## Per-Bundle NoVisual Handoff Output

After each NoVisual reconstruction, write:

```text
Bundle/
  analysis/
    unity_no_visual_reconstruction_summary.md
    unity_no_visual_reconstruction_summary.json
    unity_shader_compile_report.json
    unity_material_restore_report.json
    unity_handoff_for_visual_validation.md
```

`unity_handoff_for_visual_validation.md` must include:

- Material name and UE material path.
- Bundle path.
- Unity `.mat` path.
- Generated or reused Unity shader path.
- Shader reuse decision and reuse key.
- Restored parameter count.
- Missing or unresolved texture list.
- Texture payload source: bundle, shared, original game export, or missing.
- Implemented Master/Layer/Blend modules.
- Approximate or omitted features.
- Compile/import result.
- Known limitations.
- Recommended next visual validation goal.
- Expected RenderDoc capture location: `Bundle/RenderDocCapture`.

This file is the handoff entrypoint for later visual validation and effect alignment.

## Exporter Changes

### AgentWorkspaceWriter

1. Keep generating workspace-root bare goal launchers only when the parent directory contains exactly one `*.bundle`.
2. In multi-bundle workspaces, do not generate ambiguous root launchers.
3. Add generated docs explaining:
   - Single-bundle mode allows bare `/goal <file>.md`.
   - Multi-bundle mode requires `Bundle=...`, explicit `Bundle\Goal.md`, or a workspace batch goal.
   - RenderDoc captures belong under the matching bundle.
   - Shared texture payloads may live in `../Textures`.

### Texture Payload Export

1. Add `--texture-payload-mode`.
2. Add `--texture-root <path>` for shared mode, defaulting to `<bundleParent>/Textures`.
3. Export shared payload files only when the dedup key is missing.
4. Update bundle manifests to record texture payload mode and resolved payload references.
5. Update material restore logic to search texture sources in this order:
   - Bundle-local texture payload.
   - Shared workspace `Textures`.
   - Existing Unity project assets.
   - Explicit missing texture export from original game data.

### Batch Goal Generation

1. Add canonical repo doc:

```text
Doc/UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md
```

2. For multi-bundle workspace exports, optionally generate workspace-root:

```text
UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md
```

3. Write `batch_manifest.json` summarizing bundles, material paths, texture store mode, and status.

## Tooling Changes

### Batch Orchestrator

Add a CLI or script wrapper, for example:

```text
tools/shader_reconstruction/workspace_batch_no_visual.py
```

Responsibilities:

- Load `MaterialMap.json`.
- Resolve bundles.
- Validate no ambiguous bundle names.
- Run existing bundle-local goals or equivalent CLI commands.
- Serialize Unity writes.
- Capture command, exit code, stdout/stderr byte counts, key report paths.
- Write workspace-level batch report:

```text
batch_reports/no_visual_batch_report.json
batch_reports/no_visual_batch_report.md
```

### Restore Tool Updates

Update material restore to understand:

- `texturePayloadMode=bundle`
- `texturePayloadMode=shared`
- `workspaceTextureRoot`
- bundle-local and shared manifests

Missing texture export should write into:

- Shared `Textures` when the bundle uses shared mode.
- Bundle `texture_payload` when the bundle uses bundle mode.

## Documentation Changes

Update:

```text
Doc/UE_Cooked_Material_Bundle_Export_Goal.md
Doc/UE_Cooked_Material_Bundle_With_TexturePayload_Export_Goal.md
Doc/UE_Unity_Material_Property_Restore_Goal.md
Doc/UE_RenderDoc_Compact_Summary_Goal.md
Doc/UE_Unity_Material_OneClick_NoVisual_Reconstruction_Goal.md
```

Document:

- Single-bundle vs multi-bundle mode.
- Why bare goal filenames are disabled in multi-bundle workspaces.
- New RenderDoc capture location.
- Shared texture payload default.
- Bundle-local self-contained mode.
- Per-bundle NoVisual handoff summary.
- Batch NoVisual execution and serial Unity write policy.

## Verification Plan

### Unit/CLI Checks

1. Export one bundle in single-bundle mode.
   - Verify root bare goal launchers are generated.
   - Verify bundle is self-contained when `--texture-payload-mode bundle`.

2. Export two bundles into one workspace.
   - Verify no ambiguous bundle-local root launchers are generated.
   - Verify workspace batch goal and batch manifest are generated.

3. Export shared textures.
   - Verify duplicate texture payloads are written once.
   - Verify each bundle manifest points to the shared payload path.

4. Run material restore DryRun.
   - Verify it finds bundle-local texture payloads.
   - Verify it finds shared workspace texture payloads.
   - Verify missing texture report stays correct.

5. Run batch NoVisual with two bundles.
   - Verify Unity writes are serialized.
   - Verify each bundle receives handoff summary files.
   - Verify workspace batch report lists success/failure per bundle.

### Manual Acceptance

1. From a single-bundle workspace, run:

```text
/goal UE_Unity_Material_OneClick_NoVisual_Reconstruction_Goal.md -mat <Mat>
```

Expected: resolves automatically.

2. From a multi-bundle workspace, run:

```text
/goal UE_Unity_Material_OneClick_NoVisual_Reconstruction_Goal.md -mat <Mat>
```

Expected: stops and asks for `Bundle=...`.

3. From a multi-bundle workspace, run:

```text
/goal MI_A.bundle\UE_Unity_Material_OneClick_NoVisual_Reconstruction_Goal.md -mat <Mat>
```

Expected: processes only `MI_A.bundle`.

4. From a multi-bundle workspace, run:

```text
/goal UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md MatMap=MaterialMap.json
```

Expected: sequentially processes all mapped bundles and writes per-bundle handoff summaries.

## Migration Policy

- Existing single-bundle exports remain valid.
- Existing workspace-root RenderDocCapture directories remain readable as legacy input, but new exports and docs should use `Bundle/RenderDocCapture`.
- Existing bundle-local texture payloads remain supported.
- New multi-bundle exports default to shared `Textures`.
- New single-bundle distributable exports should explicitly use bundle texture payload mode.

## Risks

1. Shared texture manifests can become stale if files are moved manually.
   - Mitigation: verify shared payload paths and hashes before restore.

2. Batch NoVisual may hide individual material failures if reports are too compact.
   - Mitigation: per-bundle handoff report plus workspace batch report with explicit status.

3. Multiple Agents may still try to write the same Unity project.
   - Mitigation: docs must state that Unity writes are serialized unless using isolated project copies.

4. Bundle distribution may break when shared textures are not included.
   - Mitigation: handoff report must state texture payload mode and whether the bundle is self-contained.

## Implementation Phases

### Phase 1: Documentation and Mode Boundaries

- Add this plan.
- Update canonical goals for single-bundle vs multi-bundle mode.
- Update generated bundle docs to explain explicit bundle path requirements.

### Phase 2: Shared Texture Payload Store

- Implement `--texture-payload-mode`.
- Implement shared `Textures` manifest and dedup.
- Update material restore lookup logic.

### Phase 3: RenderDoc Bundle-Local Capture Layout

- Update RenderDoc compact summary goal and analyzer auto-detection.
- Prefer `Bundle/RenderDocCapture`.
- Keep legacy workspace-root detection with strict ambiguity checks.

### Phase 4: Batch NoVisual Goal and Orchestrator

- Add workspace batch goal.
- Implement batch manifest and MaterialMap handling.
- Serialize Unity write operations.
- Generate workspace-level batch report.

### Phase 5: Per-Bundle Handoff Reports

- Generate NoVisual summary JSON/Markdown.
- Generate visual validation handoff Markdown.
- Mirror key output paths into `agent_context.json`.

### Phase 6: Verification and Regression Tests

- Test single-bundle mode.
- Test multi-bundle mode.
- Test shared texture dedup.
- Test batch NoVisual.
- Test handoff summary consumption in a fresh Agent session.
