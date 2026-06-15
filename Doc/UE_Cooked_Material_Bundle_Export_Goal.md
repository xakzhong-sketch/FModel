# Goal: Export UE Cooked Material Shader Bundle

Objective: use `CUE4Parse.ShaderBundleExporter` in `D:\Github\FModel` to export one UE cooked material shader bundle, or a batch of bundles from a Unity `.mat` directory, verify the result, and leave an AI-friendly workspace for later Unity shader reconstruction.

Do not reconstruct a Unity shader in this task. Only export, refresh semantic analysis, generate Agent docs, generate batch mapping when applicable, and verify the bundle/workspace.

This is the lightweight first-step export. It includes shader data, material parameters, texture references, and cooked texture metadata, but it does not include decoded Unity-importable texture image payload files.

If the bundle will be distributed to users or Agents without original UE cooked game data and missing Unity textures must be recoverable from the bundle itself, use the payload export Goal instead:

```text
/Goal D:\Github\FModel\Doc\UE_Cooked_Material_Bundle_With_TexturePayload_Export_Goal.md 当前目录导出 MI_Name 材质
```

## How To Invoke This Goal

Preferred forms:

```text
/Goal D:\Github\FModel\Doc\UE_Cooked_Material_Bundle_Export_Goal.md 导出 MI_CG_RockSmooth_01a 材质

/Goal D:\Github\FModel\Doc\UE_Cooked_Material_Bundle_Export_Goal.md 当前目录导出 MI_CG_RockSmooth_01a 材质

/Goal D:\Github\FModel\Doc\UE_Cooked_Material_Bundle_Export_Goal.md 导出 /Game/Materials/_Master/Master/M_Character_Teeth

/Goal D:\Github\FModel\Doc\UE_Cooked_Material_Bundle_Export_Goal.md 导出 M_Character_Teeth 材质

/Goal D:\Github\FModel\Doc\UE_Cooked_Material_Bundle_Export_Goal.md Material=/Game/Art/Environment/Biome/CoralGarden/Rocks/Material/MI_CG_RockSmooth_01a Out=K:\WorkSpace\SR1\MI_CG_RockSmooth_01a.bundle

/Goal D:\Github\FModel\Doc\UE_Cooked_Material_Bundle_Export_Goal.md UnityMatDir=K:\WorkSpace\Project_Dive\Assets\Art\Environment
```

Argument rules:

```text
At the very start, before changing directories, run `Get-Location` and record that starting directory as ROOT_DIR.

If ROOT_DIR is a user workspace directory, for example:
  K:\WorkSpace\SR1
  K:\WorkSpace\ShaderReverse\MI_CG_RockSmooth_01a
  use it as the output root even if the directory name does not match the material name.

If ROOT_DIR is D:\Github\FModel or another tool/repo directory and the user did not provide Out=...:
  ask for the intended output directory.
  do not fall back to D:\ShaderWP.

If the user provides a full /Game/... material path:
  use it directly as --material.

If the user provides only a material asset name, for example M_Character_Teeth:
  pass the asset name directly to --material.
  CUE4Parse.ShaderBundleExporter performs strict built-in resolution against mounted files.
  do not write temporary resolver scripts.

If multiple matching material paths exist:
  the exporter fails and prints exact /Game/... candidates.
  report those candidates and ask the user which one to export.
  do not choose by path length or other heuristics.

If no matching path can be found:
  report that the material path could not be resolved and ask for the full /Game/... path.

If the user provides Out=...:
  use that as --out.

If the user provides UnityMatDir=...:
  this is batch export mode.
  record ROOT_DIR as the batch output root.
  recursively scan UnityMatDir for .mat files.
  the exporter maps each Unity .mat Assets-relative path to a UE /Game material path.
  Example: Assets/Art/Environment/Biome/CoralGarden/Rocks/Material/MI_CG_RockPebbles_01a.mat maps to /Game/Art/Environment/Biome/CoralGarden/Rocks/Material/MI_CG_RockPebbles_01a.
  pass UnityMatDir to the exporter as --unity-mat-dir and ROOT_DIR as --out-root.
  do not ask the user to prepare MaterialMap.json.
  after export succeeds, the batch root must contain MaterialMap.json, summary.json, batch_manifest.json, and UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md.
  later batch NoVisual reconstruction should use `/goal UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md` from ROOT_DIR.

If the user does not provide Out=...:
  if ROOT_DIR is a user workspace directory, write to ROOT_DIR\<MaterialAssetName>.bundle.
  if ROOT_DIR cannot be determined or is a tool/repo directory, stop and ask for Out=....
  never use D:\ShaderWP unless the user explicitly passed Out=D:\ShaderWP\....

If ROOT_DIR already contains exactly one *.bundle directory and its name matches the requested material asset name:
  use that existing bundle path as OUTPUT_BUNDLE.

If ROOT_DIR contains multiple *.bundle directories:
  list the candidates and ask the user to provide Out=....
  do not generate or use bare bundle-local /goal <GoalFile>.md launchers from ROOT_DIR.
  later per-material work must use Bundle\Goal.md, Bundle=..., or UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md.

If ROOT_DIR contains a RenderDocCapture directory or any RenderDoc drawcall export:
  do not process it during this cooked bundle export step.
  after the bundle export succeeds, move/copy new RenderDoc drawcall exports under OUTPUT_BUNDLE\RenderDocCapture before running compact summary.
```

Batch UnityMatDir rules:

```text
If UnityMatDir is present:
  ignore Material=... and Out=... unless the user explicitly asks for a single bundle export instead.
  require ROOT_DIR to be a user workspace directory.
  require UnityMatDir to be inside a Unity Assets directory so Assets-relative paths can be derived.
  do not write the batch output into D:\Github\FModel.
  pass --include-texture-payload --texture-payload-mode shared unless the user explicitly asks for no payload or bundle-local payload.
  duplicate .mat file names are allowed when their Assets-relative paths are different.
  when duplicate .mat file names exist, use the Assets-relative path to derive the exact /Game path and use a path-derived bundle folder name to avoid overwriting.
  if the derived /Game path does not exist in cooked UE data, report that Unity .mat as not found.
  do not fall back to same-name search.
  even if cooked UE data contains another material with the same asset name under a different relative path, treat it as not matching this Unity .mat.
```

## Project Directory

At the start of this goal, before `cd D:\Github\FModel`, record the starting shell directory as:

```text
ROOT_DIR
```

Then start the exporter from the FModel repo:

```powershell
cd D:\Github\FModel
```

Exporter project:

```text
CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj
```

## Required Inputs

For Subnautica2, use:

```text
Game:
Subnautica2

Paks:
D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\Subnautica2\Content\Paks

Mapping:
D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\5.6.1-114707+++Project+SN2-Release-Hotfix-Live-Subnautica2.usmap

decompress_shader:
D:\Github\UEShaderMapExtractor\Build\decompress_shader.exe
```

The user should provide:

```text
Material path:
/Game/...

Output bundle:
ROOT_DIR\<MaterialName>.bundle

Do not use D:\ShaderWP unless the user explicitly requested it with Out=....
```

If the user does not provide an output path and `ROOT_DIR` is available, create one under:

```text
ROOT_DIR\<MaterialName>.bundle
```

If no `ROOT_DIR` is available, do not export. Ask the user for an explicit output path:

```text
Out=...
```

Use a simple material name from the last segment of the material path.

## Export Command

Replace `MATERIAL_PATH` and `OUTPUT_BUNDLE`:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --game Subnautica2 `
  --paks "D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\Subnautica2\Content\Paks" `
  --mapping "D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\5.6.1-114707+++Project+SN2-Release-Hotfix-Live-Subnautica2.usmap" `
  --material "MATERIAL_PATH" `
  --out "OUTPUT_BUNDLE" `
  --include-layer-stack `
  --include-master-modules `
  --decompress-shader "D:\Github\UEShaderMapExtractor\Build\decompress_shader.exe" `
  --overwrite `
  --verbose
```

Example:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --game Subnautica2 `
  --paks "D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\Subnautica2\Content\Paks" `
  --mapping "D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\5.6.1-114707+++Project+SN2-Release-Hotfix-Live-Subnautica2.usmap" `
  --material "/Game/Materials/_Master/Master/M_Character_Teeth" `
  --out "K:\WorkSpace\SR1\M_Character_Teeth.bundle" `
  --include-layer-stack `
  --include-master-modules `
  --decompress-shader "D:\Github\UEShaderMapExtractor\Build\decompress_shader.exe" `
  --overwrite `
  --verbose
```

Example when this goal was started from `K:\WorkSpace\SR1`:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --game Subnautica2 `
  --paks "D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\Subnautica2\Content\Paks" `
  --mapping "D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\5.6.1-114707+++Project+SN2-Release-Hotfix-Live-Subnautica2.usmap" `
  --material "MI_CG_RockSmooth_01a" `
  --out "K:\WorkSpace\SR1\MI_CG_RockSmooth_01a.bundle" `
  --include-layer-stack `
  --include-master-modules `
  --decompress-shader "D:\Github\UEShaderMapExtractor\Build\decompress_shader.exe" `
  --overwrite `
  --verbose
```

## Batch Export From Unity .mat Directory

If the user supplied `UnityMatDir=...`, replace `UNITY_MAT_DIR` and use `ROOT_DIR` as `OUTPUT_ROOT`:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --game Subnautica2 `
  --paks "D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\Subnautica2\Content\Paks" `
  --mapping "D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\5.6.1-114707+++Project+SN2-Release-Hotfix-Live-Subnautica2.usmap" `
  --unity-mat-dir "UNITY_MAT_DIR" `
  --out-root "OUTPUT_ROOT" `
  --include-layer-stack `
  --include-master-modules `
  --include-texture-payload `
  --texture-payload-mode shared `
  --decompress-shader "D:\Github\UEShaderMapExtractor\Build\decompress_shader.exe" `
  --overwrite `
  --verbose
```

Expected batch outputs:

```text
OUTPUT_ROOT\MI_A.bundle
OUTPUT_ROOT\MI_B.bundle
OUTPUT_ROOT\Textures
OUTPUT_ROOT\MaterialMap.json
OUTPUT_ROOT\summary.json
OUTPUT_ROOT\batch_manifest.json
OUTPUT_ROOT\UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md
```

After batch export, verify the batch root:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --verify-only "OUTPUT_ROOT" `
  --verbose
```

Later NoVisual reconstruction can be started from `OUTPUT_ROOT` with:

```text
/goal UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md
```

## Verify Command

After export, run:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --verify-only "OUTPUT_BUNDLE" `
  --verbose
```

Expected result:

```text
Verify: OK
```

## Refresh Existing Bundle

If the bundle already exists and only semantic outputs / Agent docs need to be refreshed:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --semantic-only "OUTPUT_BUNDLE" `
  --verbose
```

Expected result:

```text
Semantic status: success
Agent workspace docs: updated
Verify: OK
```

## Generated Agent Docs Rule

Agent workspace Markdown and local skill files are exporter output, not hand-authored fixups.

Do not manually edit these files inside the exported bundle to satisfy audit or verification requirements:

```text
README.md
AGENTS.md
WORKFLOW.md
NEXT_TASK.md
PROMPT_NEXT_SESSION.md
UE_RenderDoc_Compact_Summary_Goal.md
UE_Unity_Material_Semantic_Visual_Validation_Goal.md
UE_Unity_Material_Lightweight_Visual_Smoke_Goal.md
UE_Unity_Material_OneClick_Reconstruction_Goal.md
UE_Unity_Material_OneClick_NoVisual_Reconstruction_Goal.md
skills/unity6-urp-deferred-shader-reconstruction/SKILL.md
agent_context.json
```

If any of those files are missing, stale, contain machine-local paths, or fail `--verify-only`:

```text
1. Do not patch bundle Markdown by hand.
2. Re-run full export, or run --semantic-only OUTPUT_BUNDLE / --context-only OUTPUT_BUNDLE.
3. Re-run --verify-only OUTPUT_BUNDLE.
4. If verification still fails, stop and report the exporter template/verifier failure.
```

Manual bundle edits hide exporter bugs and are not reproducible for batch distribution.

## Required Output Files

The bundle should contain:

```text
AGENTS.md
WORKFLOW.md
NEXT_TASK.md
PROMPT_NEXT_SESSION.md
UE_RenderDoc_Compact_Summary_Goal.md
UE_Unity_Material_Semantic_Visual_Validation_Goal.md
UE_Unity_Material_Lightweight_Visual_Smoke_Goal.md
agent_context.json
manifest.json
analysis/semantic_status.json
analysis/ai_context_pack.md
analysis/ai_context_pack.json
analysis/semantic_binding_map.json
analysis/texture_register_statistics.json
analysis/material_shader_metadata_probe.json
analysis/texture_channel_semantics.json
analysis/unity_deferred_reconstruction_contract.json
analysis/material_family.json
analysis/material_layer_stack.json
analysis/material_layer_parameter_bindings.json
analysis/material_static_permutation.json
analysis/unity_shader_reuse_key.json
analysis/unity_shader_reuse_candidates.json
analysis/unity_shader_assignment.json
analysis/unity_layer_reconstruction_contract.json
parameters/material_parameters.json
parameters/textures.json
shaders/*.dxil
shaders/*.dxil.ll
```

When `ROOT_DIR` contains exactly one `.bundle` directory, the exporter should also generate workspace-root goal launcher files beside that bundle:

```text
PROMPT_NEXT_SESSION.md
UE_RenderDoc_Compact_Summary_Goal.md
UE_Unity_Material_Semantic_Visual_Validation_Goal.md
UE_Unity_Material_Lightweight_Visual_Smoke_Goal.md
UE_Unity_Material_OneClick_Reconstruction_Goal.md
UE_Unity_Material_OneClick_NoVisual_Reconstruction_Goal.md
```

These launchers are convenience entrypoints only. A future Agent should be able to run `/goal <GoalFile>.md` from `ROOT_DIR`; the launcher must resolve the real bundle-local file from the only `.bundle` directory. If zero or multiple bundles exist, the Agent must ask for `Bundle=...` and must not guess.

When `ROOT_DIR` contains multiple `.bundle` directories, do not generate ambiguous workspace-root bundle-local launchers. Multi-bundle workspaces should use:

```text
/goal <Bundle>\UE_Unity_Material_OneClick_NoVisual_Reconstruction_Goal.md -mat <UnityMat>
/goal UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md
```

Batch/multi-bundle exports should also contain:

```text
batch_manifest.json
MaterialMap.json
MaterialMap.example.json
UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md
```

Optional RenderDoc compact runtime evidence, if a RenderDoc drawcall export has been summarized into the bundle:

```text
analysis/renderdoc/renderdoc_runtime_overlay.md
analysis/renderdoc/renderdoc_runtime_overlay.json
analysis/renderdoc/renderdoc_shader_match.json
analysis/renderdoc/renderdoc_texture_slot_map.json
analysis/renderdoc/renderdoc_runtime_outputs.json
```

## Agent Read Order After Export

For later Unity reconstruction, a new Agent should start with:

```text
AGENTS.md
WORKFLOW.md
UE_RenderDoc_Compact_Summary_Goal.md
UE_Unity_Material_Semantic_Visual_Validation_Goal.md
UE_Unity_Material_Lightweight_Visual_Smoke_Goal.md
analysis/ai_context_pack.md
analysis/ai_context_pack.json
analysis/reconstruction_entrypoints.json
analysis/unity_layer_reconstruction_contract.json, if present
analysis/material_layer_stack.json, if present
analysis/material_layer_parameter_bindings.json, if present
analysis/material_static_permutation.json, if present
analysis/unity_shader_reuse_key.json, if present
analysis/unity_shader_reuse_candidates.json, if present
analysis/unity_shader_assignment.json, if present
analysis/renderdoc/renderdoc_runtime_overlay.md, if present
analysis/renderdoc/renderdoc_runtime_overlay.json, if present
analysis/renderdoc/renderdoc_shader_match.json, if present
analysis/unity_deferred_reconstruction_contract.json
analysis/semantic_binding_map.json
analysis/texture_register_statistics.json
analysis/material_shader_metadata_probe.json
parameters/material_parameters.json
parameters/textures.json
```

Do not start by reading:

```text
shaders/*.dxil.ll
groups/*
logs/*
RenderDoc raw buffers / full cbuffer CSV / full disassembly, unless compact overlay is insufficient
large raw files
all shader variants
```

Raw DXIL/disassembly should only be opened for selected entrypoints or specific evidence questions.

## Unity Properties Rule

Generated bundle docs must instruct future Unity reconstruction Agents:

```text
Unity Shader Properties must strictly preserve UE material parameter definitions one-to-one.

Source files:
  analysis/material_layer_parameter_bindings.json, for Material Layer bundles
  parameters/material_parameters.json
  parameters/textures.json

Required preservation:
  UE parameter names
  parameter types
  default values
  texture references / paths
  exposed or override metadata where available

Do not:
  merge same-name UE parameters across different Association/Index StableKeys
  rename UE properties to friendlier Unity names
  merge multiple UE properties into one Unity property
  drop unused-looking parameters
  silently discard unsupported UE-only metadata

If Unity cannot represent a UE parameter or metadata field directly:
  keep the closest Unity property representation
  document the mismatch explicitly in the assumptions report

Shader reuse rule:
  read analysis/unity_shader_reuse_key.json
  read analysis/unity_shader_reuse_candidates.json
  read analysis/unity_shader_assignment.json
  if assignment Decision is reuse_existing, do not create a new Unity Shader
  parameter differences create different Unity .mat files, not different shaders
```

## RenderDoc Note

RenderDoc is optional. The default bundle export remains a pure UE cooked static workflow.

If the user has a RenderDoc current-drawcall export for the same material, generate compact runtime evidence with:

```text
/goal <Bundle>\UE_RenderDoc_Compact_Summary_Goal.md
```

Use the canonical `<FModelRepo>\Doc\UE_RenderDoc_Compact_Summary_Goal.md` only if the bundle-local file is missing or stale.

New RenderDoc drawcall exports should be placed under:

```text
OUTPUT_BUNDLE\RenderDocCapture
```

Workspace-root `RenderDocCapture` is a legacy fallback only and must not be guessed in multi-bundle workspaces.

Preferred output location:

```text
OUTPUT_BUNDLE\analysis\renderdoc
```

After adding or refreshing RenderDoc compact evidence, refresh the bundle Agent docs:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --context-only "OUTPUT_BUNDLE" `
  --verbose
```

Then the normal Unity reconstruction flow remains the same:

```text
cd OUTPUT_BUNDLE
/Goal PROMPT_NEXT_SESSION.md
```

After the Unity shader is reconstructed and a Unity `.mat` using that shader exists, restore cooked UE material values with:

```text
Doc/UE_Unity_Material_Property_Restore_Goal.md
```

That restore step should use `--layer-aware` when `analysis/material_layer_parameter_bindings.json` exists. It reads StableKey-derived Unity property names first, falls back to legacy name matching only when the layer-aware map is missing, writes an audit report, and only modifies the `.mat` when explicitly run with `--apply`.

After material restore succeeds with explicit `Apply`, choose the post-restore validation goal from the available evidence.

When the workspace has original-game references under `VisualRefs` / `ScreenShot*`, or the user requests reference-based semantic validation, run:

```text
/goal UE_Unity_Material_Semantic_Visual_Validation_Goal.md
```

When there are no references but the user has opened the Unity scene containing the target object and centered it in GameView, run:

```text
/goal UE_Unity_Material_Lightweight_Visual_Smoke_Goal.md
```

These are separate post-restore steps. Future exported bundle directories must include both local goal files. Semantic validation must read compact semantic reports/advice before raw screenshots or diff images, prefer decoded semantic GBuffer/material channel captures over raw GBuffer attachments, and must not create, assign, inspect, or restore Unity `.mat` files. Lightweight smoke validation is no-reference validation: it may fail obvious rendering errors or contradictions between implemented shader features and the current Unity render, but it must not claim high-fidelity UE visual parity.

`PROMPT_NEXT_SESSION.md`, `AGENTS.md`, `WORKFLOW.md`, and `agent_context.json` must support both cases:

```text
If analysis/renderdoc/renderdoc_runtime_overlay.md exists:
  read it after cooked ai_context_pack/reconstruction_entrypoints and before raw RenderDoc files.

If analysis/renderdoc/renderdoc_runtime_overlay.md does not exist:
  continue with the pure cooked static workflow.
```

`PROMPT_NEXT_SESSION.md`, `AGENTS.md`, `WORKFLOW.md`, the local skill, and `agent_context.json` must also state the Unity project access boundary:

```text
When opened through PROMPT_NEXT_SESSION.md without explicit UnityRoot=... or Mat=...:
  read only the bundle and any already-known target shader output path.
  do not recursively inspect <UnityProject>/Assets.
  do not recursively inspect any Unity Assets tree.
  do not create Unity .mat files.
  do not assign shaders to Unity .mat files.
  do not inspect or restore Unity .mat values.

Shader reuse must come from:
  analysis/unity_shader_assignment.json
  analysis/unity_shader_reuse_candidates.json
  analysis/unity_shader_reuse_key.json

If reuse evidence is missing or stale:
  ask for UnityRoot=... or a refreshed registry.
  do not scan Unity Assets to discover candidates.

Only read Unity .mat files or texture asset folders in the separate material restore step:
  Doc\UE_Unity_Material_Property_Restore_Goal.md

After material restore Apply, optional post-restore validation lives in:
  UE_Unity_Material_Semantic_Visual_Validation_Goal.md
  UE_Unity_Material_Lightweight_Visual_Smoke_Goal.md
  analysis/unity_visual_validation/visual_validation_report.md
  analysis/unity_visual_validation/visual_validation_report.json
  analysis/unity_visual_validation/visual_validation_advice.json

Semantic validation compact reports/advice must be read before raw screenshot/capture/diff files. Decoded semantic captures such as albedo.png, normal_world.png, metallic.png, smoothness.png, and occlusion.png are the primary Deferred surface evidence; raw_gbuffer0/1/2.png are debugging evidence only. Lightweight smoke validation can use the current GameView and optional semantic captures to catch obvious feature/render contradictions when no reference images exist.
```

## Completion Criteria

The task is complete only when:

```text
1. Export command finishes without exporter failure.
2. OUTPUT_BUNDLE exists.
3. manifest.json exists.
4. analysis/semantic_status.json exists and reports success.
5. analysis/texture_register_statistics.json exists.
6. If analysis/material_family.json reports HasMaterialLayers=true, all layer-aware files exist:
   analysis/material_layer_stack.json
   analysis/material_layer_parameter_bindings.json
   analysis/material_static_permutation.json
   analysis/unity_shader_reuse_key.json
   analysis/unity_shader_assignment.json
   analysis/unity_layer_reconstruction_contract.json
7. AGENTS.md and WORKFLOW.md exist in the bundle root.
8. AGENTS.md, WORKFLOW.md, NEXT_TASK.md, PROMPT_NEXT_SESSION.md, UE_RenderDoc_Compact_Summary_Goal.md, UE_Unity_Material_Semantic_Visual_Validation_Goal.md, UE_Unity_Material_Lightweight_Visual_Smoke_Goal.md, UE_Unity_Material_OneClick_Reconstruction_Goal.md, UE_Unity_Material_OneClick_NoVisual_Reconstruction_Goal.md, and the local skill mention the Unity Properties preservation rule, the Unity project access boundary, the optional RenderDoc compact summary step, and the optional post-restore validation steps where applicable.
9. --verify-only returns Verify: OK.
10. No bundle Agent Markdown, local skill file, or agent_context.json contains machine-local paths such as D:\..., K:\..., or C:\....
11. No generated bundle Agent docs were manually patched; they came from full export, --semantic-only, or --context-only.
12. Final response reports the material path, output bundle path, and verification result.
```

If any step fails, report the exact failed command, exit result, and the missing or invalid file.
