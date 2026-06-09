# Goal: Export UE Cooked Material Shader Bundle

Objective: use `CUE4Parse.ShaderBundleExporter` in `D:\Github\FModel` to export one UE cooked material shader bundle, verify it, and leave an AI-friendly workspace for later Unity shader reconstruction.

Do not reconstruct a Unity shader in this task. Only export, refresh semantic analysis, generate Agent docs, and verify the bundle.

## How To Invoke This Goal

Preferred forms:

```text
/Goal D:\Github\FModel\Doc\UE_Cooked_Material_Bundle_Export_Goal.md 导出 MI_CG_RockSmooth_01a 材质

/Goal D:\Github\FModel\Doc\UE_Cooked_Material_Bundle_Export_Goal.md 当前目录导出 MI_CG_RockSmooth_01a 材质

/Goal D:\Github\FModel\Doc\UE_Cooked_Material_Bundle_Export_Goal.md 导出 /Game/Materials/_Master/Master/M_Character_Teeth

/Goal D:\Github\FModel\Doc\UE_Cooked_Material_Bundle_Export_Goal.md 导出 M_Character_Teeth 材质

/Goal D:\Github\FModel\Doc\UE_Cooked_Material_Bundle_Export_Goal.md Material=/Game/Art/Environment/Biome/CoralGarden/Rocks/Material/MI_CG_RockSmooth_01a Out=K:\WorkSpace\SR1\MI_CG_RockSmooth_01a.bundle
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

If the user does not provide Out=...:
  if ROOT_DIR is a user workspace directory, write to ROOT_DIR\<MaterialAssetName>.bundle.
  if ROOT_DIR cannot be determined or is a tool/repo directory, stop and ask for Out=....
  never use D:\ShaderWP unless the user explicitly passed Out=D:\ShaderWP\....

If ROOT_DIR already contains exactly one *.bundle directory and its name matches the requested material asset name:
  use that existing bundle path as OUTPUT_BUNDLE.

If ROOT_DIR contains multiple *.bundle directories:
  list the candidates and ask the user to provide Out=....

If ROOT_DIR contains a RenderDocCapture directory or any RenderDoc drawcall export:
  do not process it during this cooked bundle export step.
  after the bundle export succeeds, the user or Agent may run Doc\UE_RenderDoc_Compact_Summary_Goal.md with Root=ROOT_DIR.
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

## Required Output Files

The bundle should contain:

```text
AGENTS.md
WORKFLOW.md
NEXT_TASK.md
PROMPT_NEXT_SESSION.md
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
Doc/UE_RenderDoc_Compact_Summary_Goal.md
```

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
  do not recursively inspect K:\WorkSpace\trunk\ExportedProject\Assets.
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
8. AGENTS.md, WORKFLOW.md, NEXT_TASK.md, PROMPT_NEXT_SESSION.md, and the local skill mention the Unity Properties preservation rule and the Unity project access boundary.
9. --verify-only returns Verify: OK.
10. Final response reports the material path, output bundle path, and verification result.
```

If any step fails, report the exact failed command, exit result, and the missing or invalid file.
