# UE Unity Missing Texture Export Restore Tasks

本文档把 `Doc/UE_Unity_Missing_Texture_Export_Restore_Plan.md` 拆解成可执行任务。

目标：

```text
在 Unity 材质属性恢复阶段，如果 DryRun 发现 MissingTextureGuids，则可显式从 UE cooked 游戏数据中只导出当前材质缺失贴图，等待 Unity import 生成 .meta 后，再二次 restore 绑定 GUID。
```

非目标：

```text
不在 PROMPT_NEXT_SESSION.md / shader reconstruction 阶段导出贴图。
不导出全游戏贴图。
不在导出贴图时修改 Unity .mat。
不默认覆盖已有 Unity 贴图资产。
不在第一版中强制生成 Unity .meta。
```

## Task Status Legend

```text
todo       尚未开始
doing      正在实现
blocked    需要外部信息或前置任务
done       已完成并验证
```

## Phase 0: Scope And Contracts

### T0.1 Define Restore-Stage Boundary

Status: done

Files:

```text
Doc/UE_Unity_Material_Property_Restore_Goal.md
Doc/UE_Unity_Missing_Texture_Export_Restore_Plan.md
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/AgentWorkspaceWriter.cs
```

Implementation notes:

```text
1. State that ExportMissingTextures belongs only to material restore.
2. State that PROMPT_NEXT_SESSION.md must not export textures.
3. State that texture export must not Apply .mat changes.
4. State that texture export is explicit opt-in.
```

Acceptance:

```text
Fresh Agent running bundle/PROMPT_NEXT_SESSION.md does not export textures.
Fresh Agent running UE_Unity_Material_Property_Restore_Goal.md can discover the optional ExportMissingTextures path.
```

### T0.2 Define MissingTextureExportCandidates Schema

Status: done

Output:

```text
unity_material_restore_*.json -> MissingTextureExportCandidates
```

Schema fields:

```json
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
```

Acceptance:

```text
Every MissingTextureGuids entry that has a UE ObjectPath also has a MissingTextureExportCandidates entry.
Candidates are deduplicated by ObjectPath + TextureName.
Candidates include enough data for the C# exporter to resolve cooked packages without guessing.
```

## Phase 1: Python Restore Report Upgrade

### T1.1 Add Candidate Extraction From Layer-Aware Bindings

Status: done

File:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Tools/unity_material_apply_ue_params.py
```

Implementation notes:

```text
1. During layer-aware requested property generation, keep source binding info for texture properties.
2. When resolve_texture_guid fails, emit a MissingTextureExportCandidate.
3. Pull TextureName and ObjectPath from binding.TextureName, binding.ObjectPath, binding.Value.ObjectName, and binding.Value.ObjectPath.
4. Keep StableKey and UnityName/Property in the candidate.
```

Acceptance:

```text
Layer-aware MI bundles produce missing texture candidates with StableKey and Unity property names.
No .mat is modified in dry-run mode.
```

### T1.2 Add Candidate Extraction From Legacy Texture Parameters

Status: done

File:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Tools/unity_material_apply_ue_params.py
```

Implementation notes:

```text
1. For non-layer-aware restore, generate candidates from parameters/textures.json TextureParameters.
2. Preserve ParameterName, TextureName, ObjectPath, and VirtualTextureLayerIndex.
3. Keep behavior compatible with existing MissingTextureGuids report fields.
```

Acceptance:

```text
Non-layer-aware bundles still report export candidates for missing texture GUIDs.
Existing restore reports remain parseable by older consumers.
```

### T1.3 Add Import Intent Heuristics

Status: done

File:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Tools/unity_material_apply_ue_params.py
```

Rules:

```text
BC / BCM / BaseColor / Albedo:
  ImportIntent = BaseColorOrBCM
  ColorSpace = sRGB

NR / NRO / NRH / Normal:
  ImportIntent = NormalOrPackedNormal
  ColorSpace = Linear

Mask / ORM / RMA / OAE / Packed:
  ImportIntent = MaskOrPacked
  ColorSpace = Linear

Curve / Gradient / Atlas:
  ImportIntent = CurveOrGradient
  ColorSpace = Linear unless stronger metadata says otherwise
```

Acceptance:

```text
Each MissingTextureExportCandidate has ImportIntent and ColorSpace.
Report labels these as heuristics, not proven source semantics.
```

### T1.4 Add TextureExportCommandHint

Status: done

File:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Tools/unity_material_apply_ue_params.py
```

Output:

```json
{
  "TextureExportNeeded": true,
  "TextureExportCommandHint": "dotnet run --project ... -- --export-missing-unity-textures ..."
}
```

Acceptance:

```text
When MissingTextureExportCandidates is non-empty, report includes a command hint.
When no missing textures exist, TextureExportNeeded=false or is absent.
```

## Phase 2: CLI And Options

### T2.1 Add ExportMissingUnityTextures Options

Status: done

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/ShaderBundleExportOptions.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Program.cs
```

New flags/options:

```text
--export-missing-unity-textures
--bundle <bundle>
--restore-report <restore-dryrun-report.json>
--unity-assets-root <UnityProject>/Assets
--texture-out <Assets-relative-folder>
--overwrite-textures
```

Validation rules:

```text
--export-missing-unity-textures requires --game, --paks, --mapping, --bundle, --restore-report, --unity-assets-root, --texture-out.
--texture-out must be relative to Assets or start with Assets/.
--restore-report must exist and be parseable JSON.
```

Acceptance:

```text
CLI help documents the new mode.
Invalid argument combinations fail before mounting paks.
```

### T2.2 Wire Program Dispatch

Status: done

File:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Program.cs
```

Implementation notes:

```text
1. If --export-missing-unity-textures is present, call MissingUnityTextureExporter.Export(options).
2. Do not run normal shader bundle export.
3. Do not verify shader bundle as a replacement for texture export report.
4. Return non-zero when any required texture export fails unless explicitly allowed later.
```

Acceptance:

```text
dotnet run ... -- --export-missing-unity-textures --help-like invalid args reports missing required options.
Normal bundle export behavior is unchanged.
```

## Phase 3: C# Missing Texture Exporter

### T3.1 Implement MissingUnityTextureExporter Skeleton

Status: done

New file:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/MissingUnityTextureExporter.cs
```

Responsibilities:

```text
1. Read restore dry-run report.
2. Read MissingTextureExportCandidates.
3. Deduplicate by ObjectPath.
4. Mount paks through ProviderFactory.
5. Export each missing texture to Unity Assets.
6. Write unity_missing_texture_export_report.json.
```

Acceptance:

```text
Exporter can run with a report containing zero candidates and writes a success report with no exports.
Exporter never modifies .mat files.
```

### T3.2 Resolve UE Texture ObjectPath

Status: done

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/MissingUnityTextureExporter.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/Semantic/SemanticUtilities.cs, if helper reuse is needed
```

Supported inputs:

```text
Texture2D'/Game/A/B/T_Name.T_Name'
/Game/A/B/T_Name.0
/Game/A/B/T_Name
```

Resolve to:

```text
Content/A/B/T_Name.uasset
```

Acceptance:

```text
All ObjectPath forms resolve to the same cooked package when present.
Missing packages are reported per texture and do not crash the whole report without context.
```

### T3.3 Decode And Export Texture Pixels

Status: done

File:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/MissingUnityTextureExporter.cs
```

Implementation notes:

```text
1. Load texture export from cooked package.
2. Decode to an image format supported by available CUE4Parse texture APIs.
3. Prefer PNG for first implementation.
4. Add DDS fallback if decoded export is unavailable or packed/normal fidelity requires it.
5. Preserve alpha channel when available.
```

Acceptance:

```text
BCM, mask, curve, and normal-like textures from MI_CG_RockSmooth_01a can be exported or reported with actionable failure.
Exported files are non-zero bytes and have stable paths.
```

### T3.4 Stable Unity Output Path

Status: done

File:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/MissingUnityTextureExporter.cs
```

Path rule:

```text
UnityAssetsRoot = K:\WorkSpace\trunk\ExportedProject\Assets
TextureOut = Assets/Art/Recovered/Subnautica2
UE path = /Game/Art/Surfaces/A/T_Name

Output:
K:\WorkSpace\trunk\ExportedProject\Assets\Art\Recovered\Subnautica2\Game\Art\Surfaces\A\T_Name.png
```

Acceptance:

```text
Output path mirrors UE path under TextureOut.
Existing files are skipped unless --overwrite-textures is present.
Path traversal and absolute TextureOut values are rejected.
```

### T3.5 Write Texture Sidecars

Status: done

Output:

```text
<Texture>.texture_export.json
```

Fields:

```json
{
  "TextureName": "T_Name",
  "ObjectPath": "/Game/...",
  "SourcePackage": "Content/.../T_Name.uasset",
  "OutputAsset": "Assets/...",
  "ImportIntent": "MaskOrPacked",
  "ColorSpace": "Linear",
  "NormalMap": false,
  "Evidence": []
}
```

Acceptance:

```text
Every exported image has a sidecar.
Sidecar paths are Unity-relative where useful and absolute only where needed for diagnostics.
```

### T3.6 Write Export Report

Status: done

Output:

```text
<Bundle>/unity_missing_texture_export_report.json
```

Schema:

```json
{
  "Schema": "sn2-unity-missing-texture-export/v1",
  "Bundle": "...",
  "UnityAssetsRoot": "...",
  "TextureOut": "Assets/Art/Recovered/Subnautica2",
  "Exported": [],
  "SkippedAlreadyExists": [],
  "Failed": [],
  "NextStep": "Let Unity import exported textures, then rerun restore DryRun."
}
```

Acceptance:

```text
Report is parseable JSON.
Failed items contain texture name, object path, reason, and source report entry.
Successful run with failures returns a clear non-zero or partial status policy.
```

## Phase 4: Goal Integration

### T4.1 Update Restore Goal Arguments

Status: done

File:

```text
Doc/UE_Unity_Material_Property_Restore_Goal.md
```

Add arguments:

```text
ExportMissingTextures
TextureOut=Assets/Art/Recovered/Subnautica2
OverwriteTextures
```

Rules:

```text
ExportMissingTextures does not imply Apply.
ExportMissingTextures requires a DryRun report or runs DryRun first.
Texture export must not modify .mat.
Bundle can be inferred from current directory if exactly one .bundle exists.
```

Acceptance:

```text
Goal tells fresh Agent exactly how to run dry-run -> export -> Unity import -> dry-run -> apply.
No instruction suggests doing this during shader reconstruction.
```

### T4.2 Update Human Workflow

Status: done

Files:

```text
K:\WorkSpace\ShaderReverse\Workflow.md
K:\WorkSpace\ShaderReverse\Workflow_Simple.md
```

Add concise optional step:

```text
* 如果 DryRun 报 MissingTextureGuids，可选：
  * /goal D:\Github\FModel\Doc\UE_Unity_Material_Property_Restore_Goal.md Mat=... ExportMissingTextures
  * 等 Unity import 贴图后重新 DryRun
```

Acceptance:

```text
Human-facing workflow remains short.
It does not imply texture export during PROMPT_NEXT_SESSION.md.
```

### T4.3 Update Bundle Agent Docs Template

Status: done

File:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/AgentWorkspaceWriter.cs
```

Implementation notes:

```text
1. Keep PROMPT_NEXT_SESSION.md texture-export-free.
2. Add restore-stage note that missing Unity texture GUIDs can be handled by UE_Unity_Material_Property_Restore_Goal.md ExportMissingTextures.
3. Agent docs must not tell shader reconstruction Agent to read Unity texture folders.
```

Acceptance:

```text
New bundle docs mention ExportMissingTextures only under material restore.
PROMPT_NEXT_SESSION.md remains shader/module only.
```

## Phase 5: Validation Fixtures

### T5.1 Build MI_CG_RockSmooth_01a Fixture Report

Status: done

Fixture:

```text
K:\WorkSpace\SR1\MI_CG_RockSmooth_01a.bundle
```

Expected missing examples:

```text
T_CG_RockSmooth_01a_BCM
T_CG_RockSmooth_01a_NRH
T_CG_RockSmooth_01b_BCM
T_CG_RockSmooth_01b_NRH
T_RockLichenMasks_01a
Curve_Base_Atlas
```

Acceptance:

```text
DryRun report includes MissingTextureExportCandidates for all missing textures with resolvable ObjectPath.
```

### T5.2 Run Texture Export Dry Fixture

Status: done

Command shape:

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

Acceptance:

```text
Only missing textures from the report are exported.
No .mat changes are made.
Report lists exported/skipped/failed.
```

### T5.3 Unity Import And Second Restore

Status: done

Manual/editor-dependent step:

```text
1. Let Unity import exported texture files and generate .meta files.
2. Re-run restore DryRun.
3. Confirm MissingTextureGuids count decreases or reaches zero.
```

Acceptance:

```text
DryRun can bind newly imported texture GUIDs.
Apply remains explicit and creates a .mat backup.
```

## Phase 6: Regression And Safety

### T6.1 Python Report Regression

Status: done

Command:

```powershell
python -m py_compile CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_material_apply_ue_params.py
```

Acceptance:

```text
Existing restore reports still include Matched, StableKey, LegacyFallback, Added, MissingTextureGuids, ShaderChanged.
New fields are additive.
```

### T6.2 C# Build Regression

Status: done

Command:

```powershell
dotnet build CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release
```

Acceptance:

```text
Build succeeds.
Existing --verify-only, --context-only, --semantic-only, and normal bundle export modes still work.
```

### T6.3 Safety Regression

Status: done

Checks:

```text
1. ExportMissingTextures without explicit flag never runs.
2. Texture export never modifies .mat.
3. Existing Unity texture files are skipped unless OverwriteTextures is explicit.
4. TextureOut cannot escape Unity Assets root.
5. PROMPT_NEXT_SESSION.md does not mention exporting textures during shader reconstruction.
```

Acceptance:

```text
All safety checks pass on a fresh bundle.
```

## Implementation Verification

Verified on 2026-06-09 with `MI_CG_RockSmooth_01a` fixture:

```text
Bundle:
  D:\Tmp\ShaderBundles\MI_CG_RockSmooth_01a.bundle

Unity mat:
  K:\WorkSpace\trunk\ExportedProject\Assets\Art\Environment\Biome\CoralGarden\Rocks\Material\MI_CG_RockSmooth_01a.mat

Temporary Unity Assets root:
  D:\Tmp\UnityMissingTextureExportTest2\Assets
```

Commands/results:

```text
python -m py_compile CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_material_apply_ue_params.py
  OK

dotnet build CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release --nologo -v:minimal
  Build succeeded, 0 errors.
  Note: local machine has no cmake, so CUE4Parse-Natives reports "build failed; continuing without it"; managed build still succeeds.

DryRun with empty temporary Assets root:
  Matched: 39
  StableKey: 39
  LegacyFallback: 0
  MissingTextureGuids: 3
  MissingTextureExportCandidates: 2

Legacy non-layer-aware DryRun with empty temporary Assets root:
  Matched: 15
  Added: 18
  MissingTextureGuids: 5
  MissingTextureExportCandidates: 5

Missing texture export:
  Exported: 2
  Skipped existing: 0
  Failed: 0

Second run without --overwrite-textures:
  Exported: 0
  Skipped existing: 2
  Failed: 0

Report parse:
  D:\Tmp\ShaderBundles\MI_CG_RockSmooth_01a.bundle\unity_missing_texture_export_report.json
  Status: success
  MatFilesModified: false

Sidecar parse:
  *.png.ue_texture_export.json
  Parse: OK

Second DryRun after temporary Unity-style .meta files:
  MissingTextureGuids: 0
  MissingTextureExportCandidates: 0
```

Exported texture paths preserve UE virtual package hierarchy under Unity `Assets`:

```text
Assets/Art/Recovered/Subnautica2_Test/Game/Art/Surfaces/Utility/Grunge/T_RockLichenMasks_01a.png
Assets/Art/Recovered/Subnautica2_Test/Game/Art/Surfaces/Tiling/CoralGarden/Rock/CG_RockSmooth_01a/T_CG_RockSmooth_01a_BCM.png
```

The Unity Editor import itself was not launched. The second restore pass was validated with temporary Unity-style `.meta` files because generating real `.meta` is editor-dependent and outside the exporter.

## Overall Completion Criteria

```text
1. UE_Unity_Material_Property_Restore_Goal.md documents ExportMissingTextures.
2. unity_material_apply_ue_params.py emits MissingTextureExportCandidates.
3. C# exporter has --export-missing-unity-textures mode.
4. Missing textures are exported only from the current restore report.
5. Export report and sidecars are parseable JSON.
6. No .mat is modified during texture export.
7. Re-running restore after Unity import can bind exported texture GUIDs.
8. New bundle Agent docs keep texture export out of PROMPT_NEXT_SESSION.md.
```

