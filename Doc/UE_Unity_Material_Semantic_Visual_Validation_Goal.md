# Goal: Unity Material Semantic Visual Validation

Objective: after Unity material property restore, run semantic visual validation for one reconstructed Unity 6 URP Deferred material, generate compact AI-readable reports, and use those reports to guide shader/module fixes.

Do not restore `.mat` properties in this goal. Do not export missing textures in this goal. Use `UE_Unity_Material_Property_Restore_Goal.md` for that.

## How To Invoke This Goal

Preferred forms:

```text
/goal D:\Github\FModel\Doc\UE_Unity_Material_Semantic_Visual_Validation_Goal.md

/goal D:\Github\FModel\Doc\UE_Unity_Material_Semantic_Visual_Validation_Goal.md Root=K:\WorkSpace\ShaderReverse\MI_CG_RockSmooth_01a

/goal D:\Github\FModel\Doc\UE_Unity_Material_Semantic_Visual_Validation_Goal.md Mat=K:\WorkSpace\trunk\ExportedProject\Assets\...\MI_CG_RockSmooth_01a.mat Bundle=K:\WorkSpace\ShaderReverse\MI_CG_RockSmooth_01a\MI_CG_RockSmooth_01a.bundle

/goal D:\Github\FModel\Doc\UE_Unity_Material_Semantic_Visual_Validation_Goal.md Root=K:\...\MI_xxx UnityProject=K:\WorkSpace\trunk\ExportedProject Mat=Assets\...\MI_xxx.mat
```

Argument rules:

```text
No explicit arguments:
  Treat current shell directory as Root.
  If current shell directory ends with .bundle, treat it as Bundle and use its parent as Root when the parent contains VisualRefs, ScreenShot*, or UnityValidation.

Root=...
  Material work directory containing VisualRefs, UnityValidation, and optionally one *.bundle.

Mat=...
  Unity material asset path. Prefer project-relative Assets/... path when UnityProject is provided.
  Required unless UnityValidation/config.json already contains material.

Bundle=...
  Optional cooked shader bundle. If omitted, auto-detect exactly one *.bundle under Root.

UnityProject=...
  Unity project root. Required unless UnityValidation/config.json already contains unity_project.

UnityExe=...
  Optional Unity executable. Defaults to Unity.exe or config value.

PreviewScene=...
  Optional Unity preview scene. Defaults to Assets/ShaderReverse/Validation/Scenes/SN2MaterialPreview.unity.
```

If auto-detection finds multiple bundles, list candidates and ask the user to provide `Bundle=...`.

Reference directory rules:

```text
1. Prefer Root\VisualRefs when it exists.
2. If VisualRefs is absent, `--init-config` may use an existing screenshot directory such as Root\ScreenShot, Root\Screenshots, or Root\ScreenShot01.
3. Semantic references should be PNG files named by semantic channel, for example base_color.png/albedo.png, normal_ws.png/normal_world.png, roughness.png/smoothness.png, metallic.png, ambient_occlusion.png/occlusion.png.
4. Generic screenshots such as 001.jpg or ref.jpg are treated as FinalColor only. They must not be used as proof for BaseColor, Normal, Roughness, Metallic, AO, Alpha, LayerBlend, or HeightBlend.
5. JPEG references are lossy. They are acceptable for FinalColor review, but PNG is preferred for semantic channel comparison.
```

If no usable reference directory is found, continue only far enough to create config and report `needs_reference`; do not claim visual validation passed.

## Workspace Contract

Root directory:

```text
<MaterialWorkDir>/
  VisualRefs/
  ScreenShot*/                 optional fallback final-color references
  UnityValidation/
    config.json
    captures/
      capture_manifest.json
    reports/
      visual_validation_report.json
      visual_validation_report.md
      visual_validation_advice.json
      contact_sheet.png
```

Large captures and diffs remain in `UnityValidation`. Only compact reports may be mirrored into:

```text
<Bundle>/analysis/unity_visual_validation/
```

## Required Tools

FModel repo:

```text
D:\Github\FModel
```

Python scripts:

```text
CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_visual_validation_diff.py
CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_visual_validation_advisor.py
CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_visual_validation_preflight.py
```

Unity template files:

```text
CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_shader_validation\SN2SemanticDebug.hlsl.txt
CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_shader_validation\SN2MaterialVisualValidationRunner.cs.txt
CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_shader_validation\URPMaterialSemanticCaptureFeature.cs.txt
CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_shader_validation\SN2GBufferCaptureRunner.cs.txt
CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_shader_validation\SN2GBufferCaptureRequestWatcher.cs.txt
CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_shader_validation\SN2GBufferVisualizationFeature.cs.txt
CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_shader_validation\SN2GBufferVisualize.shader.txt
```

Unity project target paths:

```text
Assets/Shaders/Subnautica2/Debug/SN2SemanticDebug.hlsl
Assets/Editor/ShaderReverse/Validation/SN2MaterialVisualValidationRunner.cs
Assets/ShaderReverse/Runtime/Validation/URPMaterialSemanticCaptureFeature.cs
Assets/Editor/ShaderReverse/Validation/SN2GBufferCaptureRunner.cs        includes request bridge
Assets/Editor/ShaderReverse/Validation/SN2GBufferCaptureRequestWatcher.cs optional standalone request bridge
Assets/ShaderReverse/Runtime/Validation/SN2GBufferVisualizationFeature.cs
Assets/ShaderReverse/Validation/Shaders/SN2GBufferVisualize.shader
Assets/ShaderReverse/Validation/gbuffer_capture_config.json   optional open-Editor config
Assets/ShaderReverse/Validation/gbuffer_capture_request.json  transient open-Editor request
Assets/ShaderReverse/Validation/gbuffer_capture_response.json transient open-Editor response
Assets/ShaderReverse/Validation/Scenes/SN2MaterialPreview.unity
```

The preview scene must contain:

```text
SN2_ValidationCamera
SN2_PreviewSphere
SN2_PreviewPlane
SN2_PreviewBeveledProxy
SN2_DirectionalLight
```

## Step 0 - Preflight

Before running Unity, generate a compact readiness report:

```powershell
python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_visual_validation_preflight.py `
  --root "ROOT_DIR" `
  --config "ROOT_DIR\UnityValidation\config.json" `
  --mat "MAT_PATH" `
  --bundle "BUNDLE_DIR" `
  --unity-project "UNITY_PROJECT" `
  --unity-exe "UNITY_EXE" `
  --out "ROOT_DIR\UnityValidation\reports" `
  --mirror-to-bundle
```

Expected output:

```text
UnityValidation\reports\visual_validation_preflight.json
UnityValidation\reports\visual_validation_preflight.md
```

Use this report to decide whether the workspace is ready for Unity capture:

```text
ready_for_unity_capture:
  Run Step 3.

blocked_unity_version:
  Do not run a mismatched Unity editor. Install/pass the exact Unity version from ProjectSettings\ProjectVersion.txt.

needs_reference:
  Add semantic PNG references under VisualRefs, or final-color screenshots under ScreenShot* for secondary review only.

capture_complete:
  Continue to diff/advisor or inspect existing compact reports.
```

## Step 1 - Generate Or Validate Config

From `D:\Github\FModel`:

```powershell
python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_visual_validation_diff.py `
  --init-config `
  --root "ROOT_DIR" `
  --mat "MAT_PATH" `
  --bundle "BUNDLE_DIR" `
  --unity-project "UNITY_PROJECT" `
  --unity-exe "UNITY_EXE" `
  --preview-scene "Assets/ShaderReverse/Validation/Scenes/SN2MaterialPreview.unity"
```

If `UnityValidation/config.json` already exists, reuse it unless the user explicitly asks to regenerate it.

## Step 2 - Install Unity Validation Templates If Needed

Copy templates into the Unity project only when missing or when the user explicitly asks to refresh them:

```powershell
Copy-Item `
  -LiteralPath "D:\Github\FModel\CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_shader_validation\SN2SemanticDebug.hlsl.txt" `
  -Destination "UNITY_PROJECT\Assets\Shaders\Subnautica2\Debug\SN2SemanticDebug.hlsl" `
  -Force

Copy-Item `
  -LiteralPath "D:\Github\FModel\CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_shader_validation\SN2MaterialVisualValidationRunner.cs.txt" `
  -Destination "UNITY_PROJECT\Assets\Editor\ShaderReverse\Validation\SN2MaterialVisualValidationRunner.cs" `
  -Force

Copy-Item `
  -LiteralPath "D:\Github\FModel\CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_shader_validation\URPMaterialSemanticCaptureFeature.cs.txt" `
  -Destination "UNITY_PROJECT\Assets\ShaderReverse\Runtime\Validation\URPMaterialSemanticCaptureFeature.cs" `
  -Force

Copy-Item `
  -LiteralPath "D:\Github\FModel\CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_shader_validation\SN2GBufferCaptureRunner.cs.txt" `
  -Destination "UNITY_PROJECT\Assets\Editor\ShaderReverse\Validation\SN2GBufferCaptureRunner.cs" `
  -Force

Copy-Item `
  -LiteralPath "D:\Github\FModel\CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_shader_validation\SN2GBufferCaptureRequestWatcher.cs.txt" `
  -Destination "UNITY_PROJECT\Assets\Editor\ShaderReverse\Validation\SN2GBufferCaptureRequestWatcher.cs" `
  -Force

Copy-Item `
  -LiteralPath "D:\Github\FModel\CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_shader_validation\SN2GBufferVisualizationFeature.cs.txt" `
  -Destination "UNITY_PROJECT\Assets\ShaderReverse\Runtime\Validation\SN2GBufferVisualizationFeature.cs" `
  -Force

Copy-Item `
  -LiteralPath "D:\Github\FModel\CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_shader_validation\SN2GBufferVisualize.shader.txt" `
  -Destination "UNITY_PROJECT\Assets\ShaderReverse\Validation\Shaders\SN2GBufferVisualize.shader" `
  -Force
```

Create parent directories first.

Do not recursively scan all `Assets`. Touch only these explicit validation script paths.

## Shader Requirements By Capture Route

There are two capture routes with different shader requirements:

```text
Step 3A - Shader semantic capture:
  Requires the reconstructed shader to expose semantic debug output.
  Use this when you need shader-internal semantic values, including Roughness, Alpha, LayerBlend, or HeightBlend.

Step 3B - URP Deferred GBuffer semantic capture:
  Does not require SN2SemanticDebug.hlsl, SN2SemanticDebug_* passes, or _SN2DebugMode.
  It decodes Unity URP Deferred GBuffer outputs from the actual renderer.
  It does require the material shader to participate in URP Deferred / UniversalGBuffer.
```

For Step 3A only, the reconstructed shader must expose semantic debug output:

```text
1. Include `SN2SemanticDebug.hlsl`.
2. Fill `SN2DebugSurface` from the same material/layer evaluation used by the UniversalGBuffer pass.
3. Expose fixed semantic passes when possible:
   SN2SemanticDebug_BaseColor
   SN2SemanticDebug_NormalWS
   SN2SemanticDebug_Roughness
   SN2SemanticDebug_Metallic
   SN2SemanticDebug_AmbientOcclusion
   SN2SemanticDebug_Alpha
4. `_SN2DebugMode` as a float material property is allowed as a backward-compatible fallback, but fixed pass names are preferred for deterministic batch capture.
```

If neither fixed semantic passes nor `_SN2DebugMode` fallback exists, `SN2MaterialVisualValidationRunner` must fail with a clear error instead of silently capturing final-color images for semantic modes.

For Step 3B and current-scene GBuffer capture, do not add shader debug passes just for the capture tool. Instead, confirm the shader has a Unity URP Deferred-compatible GBuffer path:

```text
Pass LightMode = UniversalGBuffer
URP Deferred-compatible SurfaceData / BRDFData output
BaseColor, NormalWS, Metallic/Specular, Smoothness, AO reaching Unity GBuffer
Opaque or otherwise rendered by the Deferred path
```

If the shader is Forward-only, Unlit-only, transparent-only, missing `UniversalGBuffer`, or skipped by the Deferred renderer, the GBuffer semantic captures may be black, stale, or incomplete. Fix the shader's Deferred/GBuffer pass first; do not use final lit screenshots as proof that the surface outputs are correct.

Optional raw URP GBuffer evidence:

```text
URPMaterialSemanticCaptureFeature.cs is optional.
It exposes _SN2ValidationRawGBuffer0..3 and _SN2ValidationRawDepth after URP GBuffer and writes UnityValidation/reports/urp_gbuffer_capture_contract.json.
Use this only as Unity pipeline evidence when investigating Deferred output packing or pass setup.
It does not replace semantic debug captures and it does not prove UE source material graph intent.
```

## Step 3A - Run Shader Semantic Capture

Run this when the reconstructed shader exposes `SN2SemanticDebug_*` passes or `_SN2DebugMode`.

```powershell
UNITY_EXE -batchmode -projectPath "UNITY_PROJECT" `
  -executeMethod SN2MaterialVisualValidationRunner.Run `
  -validationConfig "ROOT_DIR\UnityValidation\config.json" `
  -logFile "ROOT_DIR\UnityValidation\unity_capture.log"
```

Do not add `-nographics`; semantic RenderTexture capture needs a graphics device.

Expected output:

```text
ROOT_DIR\UnityValidation\captures\capture_manifest.json
ROOT_DIR\UnityValidation\captures\base_color.png
ROOT_DIR\UnityValidation\captures\normal_ws.png
ROOT_DIR\UnityValidation\captures\roughness.png
ROOT_DIR\UnityValidation\captures\metallic.png
ROOT_DIR\UnityValidation\captures\ambient_occlusion.png
ROOT_DIR\UnityValidation\captures\alpha.png
ROOT_DIR\UnityValidation\captures\final_color.png
```

If Unity is unavailable in this session, report that capture could not be run, but still run Python `--self-test` and leave the goal active/incomplete.

## Step 3B - Run URP Deferred GBuffer Semantic Capture

Run this when validating a Unity 6 URP Deferred material after `.mat` restore, especially when the goal is to compare Deferred surface outputs rather than only final lighting. This route does not require custom shader debug passes; it decodes Unity URP Deferred GBuffer outputs.

Before diffing GBuffer semantic captures, set the config primary semantics to Smoothness instead of Roughness:

```json
{
  "capture_modes": ["BaseColor", "NormalWS", "Smoothness", "Metallic", "AmbientOcclusion", "FinalColor"],
  "comparison": {
    "use_masks": true,
    "ignore_final_lighting_for_primary_score": true,
    "primary_semantics": ["BaseColor", "NormalWS", "Smoothness", "Metallic", "AmbientOcclusion"]
  }
}
```

If the Unity project is not open, run Unity batchmode:

```powershell
UNITY_EXE -batchmode -projectPath "UNITY_PROJECT" `
  -executeMethod SN2GBufferCaptureRunner.Run `
  -gbufferMaterial "MAT_PATH" `
  -gbufferScene "Assets/ShaderReverse/Validation/Scenes/SN2MaterialPreview.unity" `
  -gbufferOutputDir "ROOT_DIR\UnityValidation\captures_gbuffer_semantic" `
  -logFile "ROOT_DIR\UnityValidation\unity_gbuffer_capture.log"
```

Do not add `-nographics`.

If the Unity project is already open in Editor, do not launch a second Unity instance for the same project and do not use focus-stealing, SendKeys, or mouse automation. Prefer the request-bridge path.

The primary Unity-side request bridge is built into:

```text
UNITY_PROJECT\Assets\Editor\ShaderReverse\Validation\SN2GBufferCaptureRunner.cs
```

Optional standalone bridge:

```text
UNITY_PROJECT\Assets\Editor\ShaderReverse\Validation\SN2GBufferCaptureRequestWatcher.cs
```

It polls:

```text
UNITY_PROJECT\Assets\ShaderReverse\Validation\gbuffer_capture_request.json
```

and writes:

```text
UNITY_PROJECT\Assets\ShaderReverse\Validation\gbuffer_capture_response.json
```

From a generated bundle workspace, use the local command:

```powershell
.\commands\request_current_scene_gbuffer_capture.ps1 `
  -UnityProject "UNITY_PROJECT" `
  -OutputDir "ROOT_DIR\UnityValidation\captures_gbuffer_semantic_current_scene" `
  -RendererData "Assets/MonoBehaviour/URP_Renderer.asset" `
  -TimeoutSeconds 120
```

This captures the current open scene/camera without requiring Unity window focus. The Unity project must already be open, and `SN2GBufferCaptureRunner.cs` must have compiled once. The command writes `gbuffer_capture_config.json` for the current request. If Unity does not answer within a few seconds, the command touches `SN2GBufferCaptureRunner.cs` to trigger AssetDatabase refresh/script reload and writes `run_current_scene_gbuffer.flag` as a compatibility fallback for any compiled flag poller, then continues waiting. This is expected and is the supported no-focus bootstrap path.

The request command is the primary open-Editor route. Manual config/menu execution is a fallback only. The fallback config file is:

```text
UNITY_PROJECT\Assets\ShaderReverse\Validation\gbuffer_capture_config.json
```

Example:

```json
{
  "material": "Assets/.../MI_Name.mat",
  "scene": "Assets/ShaderReverse/Validation/Scenes/SN2MaterialPreview.unity",
  "output_dir": "ROOT_DIR\\UnityValidation\\captures_gbuffer_semantic",
  "use_current_scene": true,
  "capture_current_camera": false,
  "assign_material_to_preview": true
}
```

For an already-open real scene such as `Assets/Art/28_01.unity`, use current-camera mode instead. This does not require `SN2_PreviewPlane`, does not assign a material, and does not open the preview scene:

```json
{
  "output_dir": "ROOT_DIR\\UnityValidation\\captures_gbuffer_semantic",
  "renderer_data": "Assets/Settings/PC_Renderer.asset",
  "camera": "Main Camera",
  "use_current_scene": true,
  "capture_current_camera": true,
  "assign_material_to_preview": false,
  "disable_post_processing": false
}
```

Fallback only: ask the user to run this menu in the already-open Unity Editor:

```text
Tools/ShaderReverse/Validation/Capture URP Deferred GBuffer Semantics
```

Fallback only: for current real-scene validation, prefer this simpler menu:

```text
Tools/ShaderReverse/Validation/Capture Current Scene GBuffer Semantics
```

The open-Editor path has two modes:

```text
capture_current_camera=false:
  preview material mode. Uses SN2_ValidationCamera and SN2_PreviewPlane, assigning the configured/selected material.

capture_current_camera=true:
  real scene mode. Uses the configured camera, selected Camera, Main Camera, or first scene Camera. It captures the current scene as-is and does not require SN2_PreviewPlane.
```

Recommended rule for Agents:

```text
Unity not open:
  use batchmode.

Unity already open + validating a preview material:
  use commands/request_current_scene_gbuffer_capture.ps1 when validating the current scene/camera; use config + menu only as fallback.

Unity already open + validating an existing level/camera, for example Assets/Art/28_01.unity:
  use commands/request_current_scene_gbuffer_capture.ps1. Do not require Unity focus and do not launch a second Unity instance.
```

This path avoids the "Multiple Unity instances cannot open the same project" lock.

Expected output:

```text
ROOT_DIR\UnityValidation\captures_gbuffer_semantic\capture_manifest.json
ROOT_DIR\UnityValidation\captures_gbuffer_semantic\gbuffer_capture_manifest.json
ROOT_DIR\UnityValidation\captures_gbuffer_semantic\albedo.png
ROOT_DIR\UnityValidation\captures_gbuffer_semantic\normal_world.png
ROOT_DIR\UnityValidation\captures_gbuffer_semantic\metallic.png
ROOT_DIR\UnityValidation\captures_gbuffer_semantic\smoothness.png
ROOT_DIR\UnityValidation\captures_gbuffer_semantic\occlusion.png
ROOT_DIR\UnityValidation\captures_gbuffer_semantic\lit_final.png
ROOT_DIR\UnityValidation\captures_gbuffer_semantic\raw_gbuffer0.png
ROOT_DIR\UnityValidation\captures_gbuffer_semantic\raw_gbuffer1.png
ROOT_DIR\UnityValidation\captures_gbuffer_semantic\raw_gbuffer2.png
```

Use `albedo.png`, `normal_world.png`, `metallic.png`, `smoothness.png`, `occlusion.png`, and `lit_final.png` for semantic comparison. Use `raw_gbuffer0/1/2.png` only for pipeline debugging; raw attachments are not semantic material views. `normal_world.png` is produced from URP's `_CameraNormalsTexture`, so it is the NormalWS parity target; do not judge normal parity from `raw_gbuffer2.png`.

## Step 4 - Run Diff Report

```powershell
python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_visual_validation_diff.py `
  --config "ROOT_DIR\UnityValidation\config.json" `
  --captures "ROOT_DIR\UnityValidation\captures" `
  --bundle "BUNDLE_DIR" `
  --out "ROOT_DIR\UnityValidation\reports" `
  --mirror-to-bundle
```

If Step 3B was used as the primary Deferred validation route, pass the GBuffer semantic capture directory instead:

```powershell
python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_visual_validation_diff.py `
  --config "ROOT_DIR\UnityValidation\config.json" `
  --captures "ROOT_DIR\UnityValidation\captures_gbuffer_semantic" `
  --bundle "BUNDLE_DIR" `
  --out "ROOT_DIR\UnityValidation\reports" `
  --mirror-to-bundle
```

Expected output:

```text
visual_validation_report.json
visual_validation_report.md
contact_sheet.png
diff_*.png
```

`--mirror-to-bundle` copies only compact report files into bundle analysis. It does not copy large captures.

Only pass `--refs "REF_DIR"` when the user explicitly wants to override the config reference directory. Otherwise let `UnityValidation/config.json` choose `VisualRefs` or the `ScreenShot*` fallback selected during `--init-config`.

## Step 5 - Run Advisor

```powershell
python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_visual_validation_advisor.py `
  --report "ROOT_DIR\UnityValidation\reports\visual_validation_report.json" `
  --bundle "BUNDLE_DIR" `
  --out "ROOT_DIR\UnityValidation\reports"
```

If a material restore dry-run/apply report is available, pass it so missing texture GUIDs are classified as material binding issues:

```powershell
python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_visual_validation_advisor.py `
  --report "ROOT_DIR\UnityValidation\reports\visual_validation_report.json" `
  --bundle "BUNDLE_DIR" `
  --restore-report "ROOT_DIR\unity_material_restore_dryrun.json" `
  --out "ROOT_DIR\UnityValidation\reports"
```

When `--bundle` is provided, advisor also mirrors `visual_validation_advice.json` into:

```text
BUNDLE_DIR\analysis\unity_visual_validation\visual_validation_advice.json
```

## Step 6 - Refresh Bundle Agent Docs

If `Bundle=...` is available:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --context-only "BUNDLE_DIR" `
  --verbose
```

Expected:

```text
Context-only: completed
Agent workspace docs: updated
Verify: OK
```

## Step 7 - Final Preflight Summary

After diff/advisor and context refresh, run preflight again so the final report includes Root, Mat, Bundle, capture dir, report dir, status, Unity version evidence, and top issue categories:

```powershell
python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\unity_visual_validation_preflight.py `
  --root "ROOT_DIR" `
  --config "ROOT_DIR\UnityValidation\config.json" `
  --bundle "BUNDLE_DIR" `
  --restore-report "OPTIONAL_RESTORE_REPORT.json" `
  --out "ROOT_DIR\UnityValidation\reports" `
  --mirror-to-bundle
```

## Interpretation Rules

Use semantic captures as primary evidence:

```text
BaseColor
NormalWS
Roughness or Smoothness
Metallic
AmbientOcclusion
Alpha
```

For Unity URP Deferred validation, prefer decoded semantic images (`albedo`, `normal_world`, `metallic`, `smoothness`, `occlusion`) over raw GBuffer attachments. `normal_world` uses URP's camera normals texture and should be trusted over `raw_gbuffer2` for NormalWS parity. Use final lit screenshot as secondary evidence only.

If report status is:

```text
pass:
  semantic validation passed configured thresholds.

fail:
  read visual_validation_advice.json and fix shader/module logic or material binding according to issue category.

needs_reference:
  VisualRefs are missing or not semantic-comparable. Ask for references or continue with capture-only evidence.

needs_review:
  some captures or references are missing/unsupported. Fix setup before judging shader fidelity.
```

Do not treat this report as UE source material graph recovery.

## Fix Loop

If validation fails:

1. Read:

```text
UnityValidation\reports\visual_validation_report.md
UnityValidation\reports\visual_validation_report.json
UnityValidation\reports\visual_validation_advice.json
UnityValidation\reports\contact_sheet.png
```

2. Inspect only issue-specific shader/module files.
3. Do not modify `.mat` unless advice category is `material_binding`; use material restore Goal for `.mat`.
4. Re-run shader semantic capture or GBuffer semantic capture, then diff and advisor.
5. Stop only when:
   - report passes, or
   - remaining issues are documented as reference/lighting/runtime limitations.

## Completion Criteria

The task is complete only when:

```text
1. UnityValidation/config.json exists and is valid.
2. Unity capture command ran, or the final response clearly states Unity was unavailable.
3. visual_validation_report.json exists and parses.
4. visual_validation_report.md exists.
5. visual_validation_advice.json exists and parses.
6. If Bundle=... was provided, compact reports were mirrored into <Bundle>/analysis/unity_visual_validation.
7. If Bundle=... was provided, --context-only returned Verify: OK.
8. visual_validation_preflight.json/md exists and reports final workspace status.
9. Final response reports:
   - Root
   - Mat
   - Bundle
   - Capture directory
   - Report directory
   - Report status
   - Top issue categories
```

## Do Not Read By Default

```text
all raw screenshots
all Unity Assets
raw RenderDoc buffers
full shader variant dumps
```

Read raw images only when the compact report and contact sheet are insufficient.
