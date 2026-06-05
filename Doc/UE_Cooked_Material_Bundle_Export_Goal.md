# Goal: Export UE Cooked Material Shader Bundle

Objective: use `CUE4Parse.ShaderBundleExporter` in `D:\Github\FModel` to export one UE cooked material shader bundle, verify it, and leave an AI-friendly workspace for later Unity shader reconstruction.

Do not reconstruct a Unity shader in this task. Only export, refresh semantic analysis, generate Agent docs, and verify the bundle.

## How To Invoke This Goal

Preferred forms:

```text
/Goal D:\Github\FModel\Doc\UE_Cooked_Material_Bundle_Export_Goal.md 导出 /Game/Materials/_Master/Master/M_Character_Teeth

/Goal D:\Github\FModel\Doc\UE_Cooked_Material_Bundle_Export_Goal.md 导出 M_Character_Teeth 材质

/Goal D:\Github\FModel\Doc\UE_Cooked_Material_Bundle_Export_Goal.md Material=/Game/Materials/_Master/Master/M_Character_Teeth Out=D:\ShaderWP\M_Character_Teeth.bundle
```

Argument rules:

```text
If the user provides a full /Game/... material path:
  use it directly as --material.

If the user provides only a material asset name, for example M_Character_Teeth:
  first resolve it to exactly one full /Game/... material path.
  do not export until the path is resolved.

If multiple matching material paths exist:
  list the candidates and ask the user which one to export.

If no matching path can be found:
  report that the material path could not be resolved and ask for the full /Game/... path.

If the user provides Out=...:
  use that as --out.

If the user does not provide Out=...:
  write to D:\ShaderWP\<MaterialAssetName>.bundle.
```

## Project Directory

Start here:

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
D:\ShaderWP\<MaterialName>.bundle
```

If the user does not provide an output path, create one under:

```text
D:\ShaderWP\<MaterialName>.bundle
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
  --out "D:\ShaderWP\M_Character_Teeth.bundle" `
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
parameters/material_parameters.json
parameters/textures.json
shaders/*.dxil
shaders/*.dxil.ll
```

## Agent Read Order After Export

For later Unity reconstruction, a new Agent should start with:

```text
AGENTS.md
WORKFLOW.md
analysis/ai_context_pack.md
analysis/ai_context_pack.json
analysis/reconstruction_entrypoints.json
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
large raw files
all shader variants
```

Raw DXIL/disassembly should only be opened for selected entrypoints or specific evidence questions.

## RenderDoc Note

Do not use `--renderdoc-drawcall-dir` yet. The RenderDoc runtime overlay is currently planned in:

```text
Doc/UE_RenderDoc_Runtime_Overlay_Plan.md
```

Until that implementation exists, this goal is a pure UE cooked material bundle export workflow.

## Completion Criteria

The task is complete only when:

```text
1. Export command finishes without exporter failure.
2. OUTPUT_BUNDLE exists.
3. manifest.json exists.
4. analysis/semantic_status.json exists and reports success.
5. analysis/texture_register_statistics.json exists.
6. AGENTS.md and WORKFLOW.md exist in the bundle root.
7. --verify-only returns Verify: OK.
8. Final response reports the material path, output bundle path, and verification result.
```

If any step fails, report the exact failed command, exit result, and the missing or invalid file.
