# UE Unity Missing Texture Export Restore Plan

## Goal

Add an optional restore-stage workflow that exports missing Unity texture assets from the original UE cooked game data when `UE_Unity_Material_Property_Restore_Goal.md` reports `MissingTextureGuids`.

This feature belongs to material property restore, not shader reconstruction.

```text
Shader reconstruction:
  generate / extend / reuse Unity shader and modules only.

Material restore:
  create/update Unity .mat, bind shader, bind texture GUIDs, restore scalar/vector/texture values.

Missing texture export:
  optional sub-step inside material restore, after DryRun detects missing texture GUIDs.
```

## Current Problem

The Unity material restore script can restore texture references only when the texture already exists in the Unity project and has a `.meta` GUID.

Current failure pattern:

```text
DryRun report:
  MissingTextureGuids:
    T_CG_RockSmooth_01a_BCM
    T_CG_RockSmooth_01a_NRH
    T_CG_RockSmooth_01b_BCM
    T_CG_RockSmooth_01b_NRH
    T_RockLichenMasks_01a
    Curve_Base_Atlas
```

If those assets are not imported into Unity, `.mat` restore can preserve property slots but cannot bind actual texture GUIDs.

## Design Principles

1. Do not export textures during `PROMPT_NEXT_SESSION.md`.
2. Do not export all game textures.
3. Only export textures referenced by the current bundle and missing from the Unity project.
4. DryRun first; export missing textures only when explicitly requested.
5. Re-run material restore after Unity imports the exported texture files and creates `.meta` files.
6. Preserve enough metadata to set correct Unity import intent later.

## User Workflow

Normal dry-run:

```text
/goal D:\Github\FModel\Doc\UE_Unity_Material_Property_Restore_Goal.md Mat=... 
```

If `MissingTextureGuids` is non-empty, optionally export missing textures:

```text
/goal D:\Github\FModel\Doc\UE_Unity_Material_Property_Restore_Goal.md Mat=... ExportMissingTextures
```

Optional explicit output:

```text
/goal D:\Github\FModel\Doc\UE_Unity_Material_Property_Restore_Goal.md Mat=... ExportMissingTextures TextureOut=Assets/Art/Recovered/Subnautica2
```

Then let Unity import the exported files and generate `.meta` files.

Run restore again:

```text
/goal D:\Github\FModel\Doc\UE_Unity_Material_Property_Restore_Goal.md Mat=... 
```

If report looks correct:

```text
/goal D:\Github\FModel\Doc\UE_Unity_Material_Property_Restore_Goal.md Apply Mat=...
```

When invoked from a material workspace directory, `Bundle=...` remains optional if exactly one `.bundle` exists under the current directory.

## CLI / Tool Additions

### 1. Restore Script: Detect And Report Export Candidates

Extend:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Tools/unity_material_apply_ue_params.py
```

New report section:

```json
{
  "MissingTextureGuids": [],
  "MissingTextureExportCandidates": [
    {
      "Property": "_Layer0_BCM_BaseColor_Metallic_Map",
      "StableKey": "LayerParameter:0:BCM_BaseColor_Metallic_Map",
      "TextureName": "T_CG_RockSmooth_01a_BCM",
      "ObjectPath": "/Game/Art/Surfaces/Tiling/CoralGarden/Rock/CG_RockSmooth_01a/T_CG_RockSmooth_01a_BCM.0",
      "SuggestedUnityRelativePath": "Assets/Art/Recovered/Subnautica2/Game/Art/Surfaces/Tiling/CoralGarden/Rock/CG_RockSmooth_01a/T_CG_RockSmooth_01a_BCM.png",
      "ImportIntent": "BaseColorOrBCM",
      "ColorSpace": "sRGB",
      "Source": "analysis/material_layer_parameter_bindings.json"
    }
  ]
}
```

The Python restore script should not decode UE textures itself. It only identifies missing Unity GUIDs and writes candidates.

### 2. New Cooked Texture Exporter Command

Add a C# exporter mode under `CUE4Parse.ShaderBundleExporter`:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --export-missing-unity-textures `
  --game Subnautica2 `
  --paks "D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\Subnautica2\Content\Paks" `
  --mapping "D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\5.6.1-114707+++Project+SN2-Release-Hotfix-Live-Subnautica2.usmap" `
  --bundle "K:\WorkSpace\SR1\MI_CG_RockSmooth_01a.bundle" `
  --restore-report "K:\WorkSpace\SR1\MI_CG_RockSmooth_01a.bundle\unity_material_restore_dryrun.json" `
  --unity-assets-root "K:\WorkSpace\trunk\ExportedProject\Assets" `
  --texture-out "Assets/Art/Recovered/Subnautica2"
```

Required behavior:

```text
1. Read MissingTextureExportCandidates from the restore dry-run report.
2. Resolve each ObjectPath against mounted UE cooked files.
3. Export only the missing textures listed in the report.
4. Mirror the UE path under TextureOut by default.
5. Write a texture_export_report.json.
6. Never modify the Unity .mat.
7. Never recurse-export unrelated textures.
```

### 3. Optional Goal Wrapper Behavior

Update `UE_Unity_Material_Property_Restore_Goal.md` so that:

```text
ExportMissingTextures
  runs dry-run first if no recent report exists.
  reads MissingTextureGuids / MissingTextureExportCandidates.
  runs the C# texture export command.
  tells user/Agent to let Unity import textures.
  does not Apply .mat changes.
```

## Texture Source Resolution

Preferred source order:

```text
1. analysis/material_layer_parameter_bindings.json
   Texture binding Value.ObjectPath / TextureName.

2. parameters/textures.json
   TextureParameters[].ObjectPath and ReferencedTextures.

3. analysis/texture_assets.json
   Cooked texture metadata and semantic hints, if present.

4. source/textures/*.cooked.json
   Extra metadata only; not the source pixel payload.
```

ObjectPath normalization:

```text
Texture2D'/Game/A/B/T_Name.T_Name'
/Game/A/B/T_Name.0
/Game/A/B/T_Name
```

should resolve to:

```text
Content/A/B/T_Name.uasset
```

## Export Format

Preferred first implementation:

```text
PNG for ordinary color/mask/curve textures.
TGA or PNG for normal-like textures if channel preservation is safe.
```

If CUE4Parse can expose the original compressed payload more reliably than decoded pixels, support:

```text
DDS export as fallback.
```

Output should include:

```text
<TextureOut>/Game/.../T_Name.png
<TextureOut>/Game/.../T_Name.texture_export.json
```

The `.texture_export.json` sidecar should include:

```json
{
  "TextureName": "T_CG_RockSmooth_01a_BCM",
  "ObjectPath": "/Game/...",
  "SourcePackage": "Content/.../T_CG_RockSmooth_01a.uasset",
  "OutputAsset": "Assets/Art/Recovered/Subnautica2/Game/.../T_CG_RockSmooth_01a_BCM.png",
  "ImportIntent": "BaseColorOrBCM",
  "ColorSpace": "sRGB",
  "NormalMap": false,
  "CompressionHint": "Default",
  "Evidence": [
    "Texture parameter/name suggests base color or color texture",
    "analysis/texture_assets.json"
  ]
}
```

## Unity Import Intent

The exporter should not silently guess all import settings as facts. It should emit import intent.

Initial heuristics:

```text
BC / BCM / BaseColor / Albedo:
  ColorSpace = sRGB
  NormalMap = false

NR / NRO / NRH / Normal:
  ColorSpace = Linear
  NormalMap = true or packed-normal intent

Mask / ORM / RMA / OAE / Packed:
  ColorSpace = Linear
  NormalMap = false

Curve / Gradient / Atlas:
  ColorSpace = Linear unless metadata proves color lookup
  NormalMap = false
```

If Unity importer automation is added later, it should use these sidecars to set `.meta` import settings. First version can leave importer settings manual but must report the intended settings.

## Reports

### Restore DryRun Report

Add:

```json
{
  "TextureExportNeeded": true,
  "MissingTextureExportCandidates": [],
  "TextureExportCommandHint": "dotnet run ... --export-missing-unity-textures ..."
}
```

### Texture Export Report

Write:

```text
<Bundle>/unity_missing_texture_export_report.json
```

Schema:

```json
{
  "Schema": "sn2-unity-missing-texture-export/v1",
  "Bundle": "K:/WorkSpace/SR1/MI_CG_RockSmooth_01a.bundle",
  "UnityAssetsRoot": "K:/WorkSpace/trunk/ExportedProject/Assets",
  "TextureOut": "Assets/Art/Recovered/Subnautica2",
  "Exported": [],
  "SkippedAlreadyExists": [],
  "Failed": [],
  "NextStep": "Let Unity import exported textures, then rerun UE_Unity_Material_Property_Restore_Goal.md DryRun."
}
```

Each exported item:

```json
{
  "TextureName": "T_Name",
  "ObjectPath": "/Game/...",
  "OutputAsset": "Assets/...",
  "OutputFullPath": "K:/.../Assets/...",
  "ImportIntent": "MaskOrPacked",
  "ColorSpace": "Linear",
  "Status": "exported"
}
```

## Safety Rules

```text
Do not export textures during shader reconstruction.
Do not export textures unless ExportMissingTextures is explicit.
Do not apply .mat changes in the same command that exports missing textures.
Do not overwrite existing Unity texture assets unless OverwriteTextures is explicit.
Do not generate or edit .meta files in the first implementation.
Do not export all textures from the game.
Do not treat texture semantic heuristics as proof; record them as import intent.
```

## Implementation Phases

### Phase 1: Restore Report Candidates

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Tools/unity_material_apply_ue_params.py
Doc/UE_Unity_Material_Property_Restore_Goal.md
```

Tasks:

```text
1. Add MissingTextureExportCandidates to dry-run report.
2. Include ObjectPath, TextureName, StableKey, UnityName, and Source.
3. Include a command hint for the C# exporter.
4. Keep existing MissingTextureGuids behavior unchanged.
```

Validation:

```text
DryRun report for MI_CG_RockSmooth_01a lists missing BCM/NRH/mask/curve textures with ObjectPath.
No .mat is modified.
```

### Phase 2: C# Missing Texture Exporter

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/MissingUnityTextureExporter.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/ShaderBundleExportOptions.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Program.cs
```

Tasks:

```text
1. Add --export-missing-unity-textures mode.
2. Add --bundle, --restore-report, --unity-assets-root, --texture-out.
3. Mount paks with existing ProviderFactory.
4. Resolve texture ObjectPath to cooked package.
5. Decode/export texture image.
6. Write sidecar and export report.
```

Validation:

```text
Can export only missing textures listed in a dry-run report.
No unrelated texture is exported.
Existing Unity texture files are skipped by default.
```

### Phase 3: Goal Integration

Files:

```text
Doc/UE_Unity_Material_Property_Restore_Goal.md
Doc/UE_Unity_Missing_Texture_Export_Restore_Plan.md
K:\WorkSpace\ShaderReverse\Workflow_Simple.md
```

Tasks:

```text
1. Document ExportMissingTextures argument.
2. Document current-directory auto Bundle detection.
3. Document dry-run -> export -> Unity import -> dry-run -> apply sequence.
4. Explicitly state this step is separate from PROMPT_NEXT_SESSION.md.
```

Validation:

```text
Fresh Agent can run material restore from a workspace with exactly one bundle and Mat=...
Fresh Agent does not write ad hoc texture export scripts.
```

### Phase 4: Optional Unity Import Settings Automation

This is optional and should come after basic export works.

Possible implementation:

```text
Generate Unity Editor script or AssetPostprocessor that reads *.texture_export.json sidecars and applies:
  sRGB on/off
  TextureType NormalMap for normal intent
  Compression mode
```

Do not block Phase 1-3 on this.

## Acceptance Criteria

```text
1. Restore DryRun reports MissingTextureExportCandidates with ObjectPath and suggested Unity output.
2. ExportMissingTextures exports only textures missing from the current material restore report.
3. Exported texture files appear under the requested Unity Assets subfolder.
4. texture_export_report.json is parseable and lists exported/skipped/failed items.
5. No Unity .mat is modified during texture export.
6. After Unity imports textures, rerunning restore DryRun reduces or clears MissingTextureGuids.
7. Apply still requires explicit Apply and creates .mat backup.
8. PROMPT_NEXT_SESSION.md shader reconstruction remains texture-export-free.
```

## Open Questions

```text
1. Preferred file format: PNG-only first, or DDS fallback for packed/normal textures?
2. Should first implementation generate Unity .meta/import settings, or only sidecar intent?
3. Where should recovered textures live by default?
   Proposed: Assets/Art/Recovered/Subnautica2/Game/...
4. Should virtual textures be exported as ordinary Texture2D approximations, or reported separately?
```
