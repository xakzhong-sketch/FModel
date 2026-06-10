# UE Unity Material Semantic Visual Validation Design

本文档设计一个后续视觉验证工作流，用于在完成：

```text
/goal D:\Github\FModel\Doc\UE_Unity_Material_Property_Restore_Goal.md Apply Mat=... Bundle=...
```

之后，让 AI Agent 自动验证 Unity 还原材质与原游戏材质截图或 RenderDoc 语义证据是否一致，并在不一致时循环修正 Unity Shader。

核心判断：

- 人工使用 URP Rendering Debugger 很适合检查问题，但不适合作为稳定自动化流程。
- 自动化主路径应使用 Unity 侧自定义 capture 工具，直接输出 BaseColor、Normal、Roughness、Metallic、AO 等 semantic buffers。
- 只在需要最终 Deferred pipeline 校验时，再读取/解码 URP raw GBuffer。
- 最终 lit screenshot 只作为辅助验收，不作为第一修正依据。

## Goals

1. 在 Unity 中自动渲染目标材质并导出语义截图。
2. 将 Unity 语义截图与原游戏 reference、RenderDoc 输出或 UE semantic reference 做结构化对比。
3. 输出 compact report，供 AI Agent 判断差异原因。
4. 支持自动循环：capture -> diff -> shader fix -> recapture。
5. 避免 Agent 直接读取大量 raw RenderDoc、整项目 Assets 或全量截图。

## Non-Goals

- 不要求在第一版中完全复刻 UE Deferred LightPass。
- 不直接逐像素比较 UE raw GBuffer RT 与 Unity raw GBuffer RT。
- 不通过 UI 自动点击 Rendering Debugger 作为主流程。
- 不把大型 PNG/EXR 全量塞进 AI 上下文。
- 不在材质属性恢复 Goal 中直接修改 shader；shader 修正仍属于 shader reconstruction/visual fix loop。

## Validation Strategy

### Primary: Shader Semantic Capture

每个还原 Unity Shader 必须支持一个 debug semantic 输出路径。该路径直接从 shader 内部 surface/module 结果输出语义值：

```text
BaseColor
NormalWS
NormalTS
Roughness
Smoothness
Metallic
AmbientOcclusion
Emission
Alpha
OpacityMask
LayerBlendMask
HeightBlend
CustomPrimitiveDataDerivedValues
```

优点：

- 最接近 shader 逻辑本身，适合定位错误。
- 不依赖 URP GBuffer packing。
- 便于新增自定义 Layer/Blend 语义图。
- 可稳定 batchmode 导出。

### Secondary: URP Deferred Semantic/GBuffer Capture

通过 ScriptableRendererFeature 在 URP Deferred GBuffer pass 之后、Lighting 之前捕获：

```text
Unity raw GBuffer attachments
Unity decoded GBuffer semantic views
Depth
FinalColor before/after postprocess
```

用途：

- 验证 shader 的 `UniversalGBuffer` pass 是否真的写入了 URP Deferred。
- 验证 Material Debug Semantic 与实际 URP Deferred 输出是否一致。
- 排查 Unity pass/tag/keyword/variant 错误。

限制：

- URP GBuffer packing 受 Unity 版本、平台、材质模型影响。
- raw GBuffer 不应直接和 UE raw GBuffer 做一一对应比较。
- 对比时必须先解码为 semantic-level buffer。

### Fallback: URP Rendering Debugger

URP Rendering Debugger 可作为人工或 fallback 验证：

```text
Material Override:
  Albedo
  Specular
  Alpha
  Smoothness
  AmbientOcclusion
  Emission
  NormalWorldSpace
  NormalTangentSpace
  Metallic
```

用途：

- 人工快速检查。
- 验证 shader 是否正确接入 URP debug display。

不作为主自动化路径，因为 UI 状态、GameView、焦点、分辨率、Play Mode 都会影响稳定性。

## Workspace Layout

材质工作目录建议结构：

```text
<MaterialWorkDir>/
  MI_CG_RockSmooth_01a.bundle/
  RenderDocCapture/
  VisualRefs/
    ref_001.png
    ref_001.mask.png
    ref_001.meta.json
    ref_002.png
    ref_002.mask.png
    ref_002.meta.json
  UnityValidation/
    config.json
    captures/
      base_color.png
      normal_ws.exr
      roughness.png
      metallic.png
      ao.png
      final_color.png
      raw_gbuffer0.exr
      raw_gbuffer1.exr
    reports/
      visual_validation_report.json
      visual_validation_report.md
      contact_sheet.png
      diff_base_color.png
      diff_normal_ws.png
```

## Reference Input Contract

### `VisualRefs/*.meta.json`

每张原游戏截图建议配一个 metadata 文件：

```json
{
  "schema": "ue-unity-material-visual-reference/v1",
  "material": "MI_CG_RockSmooth_01a",
  "reference_image": "ref_001.png",
  "mask_image": "ref_001.mask.png",
  "region": "main_visible_rock_surface",
  "source": {
    "type": "game_screenshot|renderdoc_output|manual_crop",
    "capture": "RenderDocCapture/EID_6250_..."
  },
  "focus": [
    "base_color",
    "normal_ws",
    "roughness",
    "layer_blend",
    "lichen_mask"
  ],
  "ignore": [
    "sky",
    "background",
    "deep_shadow",
    "specular_highlight",
    "postprocess_bloom"
  ],
  "camera_hint": {
    "view": "close_up",
    "angle": "grazing",
    "distance": "near"
  },
  "notes": "Use only the masked rock surface. Do not judge lighting or background."
}
```

如果 reference 来自 RenderDoc compact summary，应优先使用：

```text
analysis/renderdoc/renderdoc_material_runtime_evidence.json
analysis/renderdoc/renderdoc_gbuffer_outputs.json
analysis/renderdoc/renderdoc_shader_io_summary.json
```

作为语义证据，而不是让 Agent 直接读 raw RenderDoc export。

## Unity Tools To Add

### 1. Shader Semantic Debug Include

Suggested path:

```text
Assets/Shaders/Subnautica2/Debug/SN2SemanticDebug.hlsl
```

职责：

- 定义 semantic debug enum/keywords。
- 提供统一编码函数。
- 将 shader 内部 `SN2MaterialSurface` 或类似结构输出为 debug color。

建议 keywords：

```hlsl
#pragma multi_compile_local_fragment _ SN2_DEBUG_BASECOLOR SN2_DEBUG_NORMALWS SN2_DEBUG_NORMALTS
#pragma multi_compile_local_fragment _ SN2_DEBUG_ROUGHNESS SN2_DEBUG_SMOOTHNESS SN2_DEBUG_METALLIC
#pragma multi_compile_local_fragment _ SN2_DEBUG_AO SN2_DEBUG_EMISSION SN2_DEBUG_ALPHA
#pragma multi_compile_local_fragment _ SN2_DEBUG_LAYER_BLEND SN2_DEBUG_HEIGHT_BLEND
```

建议输出规则：

```text
base_color: linear RGB, alpha 1
normal_ws: normal * 0.5 + 0.5
normal_ts: normal * 0.5 + 0.5
roughness: R=roughness, G=roughness, B=roughness
smoothness: R=smoothness, G=smoothness, B=smoothness
metallic: grayscale
ao: grayscale
emission: linear RGB, tone-map disabled when possible
alpha: grayscale
layer_blend: custom debug packed channels
height_blend: custom debug packed channels
```

### 2. Shader Debug Pass Or Replacement Material

Suggested implementation options:

#### Option A - Debug Pass In Every Reconstructed Shader

Add a pass:

```text
Pass "SN2SemanticDebug"
```

Tags:

```text
"LightMode" = "SRPDefaultUnlit"
```

The pass evaluates the same material module stack as the real `UniversalGBuffer` pass, then returns selected semantic output.

Pros:

- Highest fidelity for shader logic.
- Easy to add custom layer/blend debug outputs.

Cons:

- Every generated shader must include the pass.
- Need to keep real pass and debug pass code paths sharing the same module evaluation.

#### Option B - Replacement Debug Shader

Use a dedicated debug shader that reads material properties and calls shared module functions.

Pros:

- Less code in final production shader.

Cons:

- More risk of diverging from actual shader path.
- Harder to support all reconstructed modules.

Recommendation: use Option A for reconstructed shaders.

### 3. URP Semantic Capture Renderer Feature

Suggested path:

```text
Assets/ShaderReverse/Runtime/Validation/URPMaterialSemanticCaptureFeature.cs
```

Responsibilities:

- Render target material/object multiple times with different debug keywords.
- Export semantic buffers to named `RenderTexture`s.
- Optionally capture final color and raw URP GBuffer attachments.

Suggested capture modes:

```csharp
public enum SN2SemanticCaptureMode
{
    BaseColor,
    NormalWS,
    NormalTS,
    Roughness,
    Smoothness,
    Metallic,
    AmbientOcclusion,
    Emission,
    Alpha,
    LayerBlend,
    HeightBlend,
    FinalColor,
    RawGBuffer0,
    RawGBuffer1,
    RawGBuffer2,
    RawGBuffer3,
    Depth
}
```

Implementation notes:

- Unity 6 URP uses RenderGraph for custom passes.
- Keep RenderFeature optional and Editor/validation-only.
- Use stable resolution from config, for example 1024x1024 or 1920x1080.
- Disable post-processing for semantic captures.
- Use linear color space.
- Prefer EXR for normal/depth/high precision outputs; PNG is acceptable for quick reports.

### 4. Unity Editor Validation Runner

Suggested path:

```text
Assets/Editor/ShaderReverse/Validation/SN2MaterialVisualValidationRunner.cs
```

Batch method:

```text
SN2MaterialVisualValidationRunner.Run
```

Command:

```powershell
Unity.exe -batchmode -projectPath "K:\WorkSpace\trunk\ExportedProject" `
  -executeMethod SN2MaterialVisualValidationRunner.Run `
  -validationConfig "K:\WorkSpace\ShaderReverse\MI_CG_RockSmooth_01a\UnityValidation\config.json" `
  -logFile "K:\WorkSpace\ShaderReverse\MI_CG_RockSmooth_01a\UnityValidation\unity_capture.log"
```

Responsibilities:

1. Load validation config.
2. Open preview scene.
3. Assign target material to preview mesh.
4. Set camera, lighting, exposure, resolution.
5. Run semantic capture modes.
6. Save captures and manifest.
7. Return non-zero exit code only on tool failure, not visual mismatch.

### 5. Preview Scene

Suggested path:

```text
Assets/ShaderReverse/Validation/Scenes/SN2MaterialPreview.unity
```

Scene contents:

```text
PreviewSphere
PreviewPlane
PreviewBeveledRockProxy
PreviewTriplanarBox
DirectionalLight
NeutralHDRI or fixed ambient
ValidationCamera
```

Rules:

- Fixed transform/camera/lights.
- Post-processing disabled for semantic captures.
- Optional final-lit capture can use fixed URP volume.
- Mesh should include UVs, normals, tangents, vertex colors, and optional custom attributes if reconstructed shaders use them.

### 6. Visual Diff Script

Suggested path on FModel side:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Tools/unity_visual_validation_diff.py
```

Alternative Unity-side path:

```text
Assets/Editor/ShaderReverse/Validation/SN2MaterialVisualDiff.cs
```

Recommendation: implement diff in Python first because it is easier to iterate and can run outside Unity.

Inputs:

```text
--config <UnityValidation/config.json>
--captures <UnityValidation/captures>
--refs <VisualRefs>
--bundle <Material.bundle>
--out <UnityValidation/reports>
```

Outputs:

```text
visual_validation_report.json
visual_validation_report.md
contact_sheet.png
diff_*.png
```

Metrics:

```text
mean_absolute_error
rmse
ssim
masked_mean_error
histogram_delta
normal_angle_error_degrees
roughness_mean_delta
metallic_mean_delta
edge_detail_delta
```

Do not require all metrics on day one. The first useful version can implement:

```text
MAE
RMSE
masked MAE
normal angle error
contact sheet
```

### 7. Visual Fix Advisor

Suggested path:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Tools/unity_visual_validation_advisor.py
```

This can be a lightweight rules pass that turns diff metrics into likely shader issues:

```json
{
  "likely_issues": [
    {
      "type": "normal_strength_too_low",
      "evidence": ["normal_angle_error_degrees > threshold", "final highlights too flat"],
      "suggested_files": ["MaterialModules/ML_LayerStandard.hlsl"],
      "suggested_checks": ["normal decode", "normal intensity", "normal blend"]
    }
  ]
}
```

This keeps the AI Agent from over-reading images. The Agent should read the report first, then inspect only the most relevant diffs.

## Validation Config

`UnityValidation/config.json`:

```json
{
  "schema": "unity-material-semantic-validation/v1",
  "unity_project": "K:/WorkSpace/trunk/ExportedProject",
  "material": "Assets/Art/Environment/Biome/CoralGarden/Rocks/Material/MI_CG_RockSmooth_01a.mat",
  "bundle": "K:/WorkSpace/ShaderReverse/MI_CG_RockSmooth_01a/MI_CG_RockSmooth_01a.bundle",
  "preview_scene": "Assets/ShaderReverse/Validation/Scenes/SN2MaterialPreview.unity",
  "output_dir": "K:/WorkSpace/ShaderReverse/MI_CG_RockSmooth_01a/UnityValidation",
  "resolution": {
    "width": 1024,
    "height": 1024
  },
  "capture_modes": [
    "BaseColor",
    "NormalWS",
    "Roughness",
    "Metallic",
    "AmbientOcclusion",
    "Emission",
    "Alpha",
    "FinalColor"
  ],
  "reference_dir": "K:/WorkSpace/ShaderReverse/MI_CG_RockSmooth_01a/VisualRefs",
  "comparison": {
    "use_masks": true,
    "ignore_final_lighting_for_primary_score": true,
    "primary_semantics": [
      "BaseColor",
      "NormalWS",
      "Roughness",
      "Metallic",
      "AmbientOcclusion"
    ]
  },
  "thresholds": {
    "base_color_mae": 0.08,
    "normal_angle_mean_degrees": 12.0,
    "roughness_mae": 0.10,
    "metallic_mae": 0.08,
    "ao_mae": 0.10
  }
}
```

## Capture Manifest

`UnityValidation/captures/capture_manifest.json`:

```json
{
  "schema": "unity-material-semantic-capture/v1",
  "unity_version": "6000.x",
  "render_pipeline": "URP",
  "rendering_path": "Deferred",
  "material": "Assets/.../MI_CG_RockSmooth_01a.mat",
  "shader": "Assets/Shaders/Subnautica2/Reconstructed/...",
  "scene": "Assets/ShaderReverse/Validation/Scenes/SN2MaterialPreview.unity",
  "camera": {
    "width": 1024,
    "height": 1024,
    "hdr": false,
    "postprocess": false
  },
  "captures": [
    {
      "semantic": "BaseColor",
      "path": "base_color.png",
      "format": "PNG",
      "color_space": "linear_encoded_for_debug",
      "source": "SN2SemanticDebug"
    },
    {
      "semantic": "NormalWS",
      "path": "normal_ws.exr",
      "format": "EXR",
      "encoding": "normal * 0.5 + 0.5",
      "source": "SN2SemanticDebug"
    }
  ]
}
```

## Diff Report

`UnityValidation/reports/visual_validation_report.json`:

```json
{
  "schema": "unity-material-visual-validation-report/v1",
  "status": "pass|fail|needs_review",
  "material": "MI_CG_RockSmooth_01a",
  "primary_score": 0.74,
  "semantic_results": [
    {
      "semantic": "BaseColor",
      "status": "fail",
      "mae": 0.14,
      "rmse": 0.19,
      "masked_mae": 0.12,
      "likely_issue": "base_color_tint_or_texture_binding"
    },
    {
      "semantic": "NormalWS",
      "status": "fail",
      "normal_angle_mean_degrees": 24.5,
      "likely_issue": "normal_decode_or_intensity"
    }
  ],
  "runtime_context": {
    "renderdoc_gbuffer_summary": "bundle/analysis/renderdoc/renderdoc_gbuffer_outputs.json",
    "renderdoc_material_runtime_evidence": "bundle/analysis/renderdoc/renderdoc_material_runtime_evidence.json"
  },
  "agent_read_first": [
    "visual_validation_report.md",
    "visual_validation_report.json",
    "contact_sheet.png"
  ],
  "do_not_read_by_default": [
    "all raw screenshots",
    "all Unity Assets",
    "raw RenderDoc buffers"
  ]
}
```

`visual_validation_report.md` should contain:

- Summary table.
- Top failing semantics.
- Small contact sheet paths.
- Likely issue classification.
- Exact files or shader modules the Agent should inspect.
- Explicit note whether failure is shader logic, texture binding, scene setup, or lighting-only.

## AI Agent Workflow

### Step 1 - Restore Material Properties

```text
/goal D:\Github\FModel\Doc\UE_Unity_Material_Property_Restore_Goal.md Apply Mat=... Bundle=...
```

This step only restores Unity `.mat` properties and optionally exports missing textures.

### Step 2 - Run Visual Validation

Future goal document:

```text
Doc/UE_Unity_Material_Semantic_Visual_Validation_Goal.md
```

Invocation examples:

```text
/goal D:\Github\FModel\Doc\UE_Unity_Material_Semantic_Visual_Validation_Goal.md Mat=... Bundle=...

/goal D:\Github\FModel\Doc\UE_Unity_Material_Semantic_Visual_Validation_Goal.md Root=K:\WorkSpace\ShaderReverse\MI_CG_RockSmooth_01a
```

The Goal should:

1. Detect `VisualRefs`.
2. Detect `Bundle`.
3. Detect `Mat`.
4. Generate or update `UnityValidation/config.json`.
5. Run Unity capture batch command.
6. Run diff script.
7. Report pass/fail and top issues.

### Step 3 - AI Fix Loop

If validation fails:

1. Read `visual_validation_report.md/json`.
2. Read `contact_sheet.png` and only the top failing `diff_*.png`.
3. Inspect shader/module files named by report.
4. Fix shader logic, not material values, unless report says property restore is wrong.
5. Re-run Unity compile/capture/diff.
6. Repeat until:
   - semantic score passes thresholds, or
   - remaining differences are documented as renderer/lighting/reference limitations.

### Step 4 - Final Acceptance

Pass criteria:

```text
Shader compiles in Unity 6 URP Deferred.
DOTS instancing still supported.
Material properties remain one-to-one with UE definitions.
Semantic captures pass configured thresholds or remaining issues are explicitly documented.
Final lit screenshot is visually plausible under fixed preview scene.
No broad Unity Assets scan was performed.
No UE LightPass logic was ported into the material shader unless separate evidence required it.
```

## Comparison Priority

Use this order:

1. Semantic shader debug outputs.
2. Unity decoded URP GBuffer outputs.
3. RenderDoc decoded UE semantic outputs, if available.
4. Masked original-game screenshots.
5. Final lit preview screenshot.

Do not make final lit screenshot the primary score unless no semantic references exist.

## Integration With Existing Bundle Evidence

The validation Agent should read bundle evidence in this order:

```text
analysis/ai_context_pack.md/json
analysis/reconstruction_entrypoints.json
analysis/module_formula_evidence.json
analysis/dxil_formula_evidence.json
analysis/curve_atlas_metadata.json
analysis/unity_layer_reconstruction_contract.json
analysis/material_layer_stack.json
analysis/material_layer_parameter_bindings.json
analysis/material_static_permutation.json
analysis/renderdoc/renderdoc_material_runtime_evidence.json
analysis/renderdoc/renderdoc_shader_io_summary.json
analysis/renderdoc/renderdoc_cpd_summary.json
analysis/renderdoc/renderdoc_svt_summary.json
analysis/renderdoc/renderdoc_gbuffer_outputs.json
UnityValidation/reports/visual_validation_report.md/json
```

RenderDoc compact evidence is runtime evidence, not UE source graph recovery.

## Error Classification

The diff/advisor should classify failures into stable categories:

```text
texture_missing_or_wrong_guid
texture_color_space_wrong
base_color_tint_wrong
normal_decode_wrong
normal_intensity_wrong
normal_blend_wrong
roughness_smoothness_inverted
metallic_channel_wrong
ao_channel_wrong
mask_channel_wrong
height_blend_wrong
vertex_color_missing
custom_primitive_data_missing
curve_gradient_wrong
uv_transform_wrong
triplanar_or_object_scale_wrong
layer_order_wrong
deferred_pass_not_writing_expected_outputs
lighting_or_postprocess_only
reference_capture_not_comparable
```

This classification matters because only some failures should cause shader edits.

## Implementation Phases

### Phase 1 - Manual-Callable Validation

Add:

```text
SN2SemanticDebug.hlsl
SN2MaterialVisualValidationRunner.cs
unity_visual_validation_diff.py
```

Require generated shaders to include semantic debug keywords/pass.

Output:

```text
UnityValidation/captures/*
UnityValidation/reports/visual_validation_report.*
```

### Phase 2 - Goal Workflow

Add:

```text
Doc/UE_Unity_Material_Semantic_Visual_Validation_Goal.md
```

The goal runs capture + diff and tells Agent how to fix shader issues.

### Phase 3 - Bundle Doc Integration

Update generated bundle docs to mention:

```text
After material restore Apply, run semantic visual validation if VisualRefs exists.
Do not use final lit screenshot alone as proof of shader fidelity.
Use visual_validation_report.md/json before reading raw screenshots.
```

### Phase 4 - URP Raw GBuffer Capture

Add optional RenderFeature pass for:

```text
RawGBuffer0..3
DecodedGBuffer/BaseColor/Normal/Roughness/Metallic/AO
Depth
```

Use this only after semantic debug pass is working.

## Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Reference screenshots include lighting/postprocess | False shader failures | Use masks/meta and semantic references first |
| Debug pass diverges from real pass | Misleading validation | Share module evaluation code between debug and UniversalGBuffer passes |
| URP GBuffer packing changes | Wrong raw GBuffer decode | Prefer semantic shader debug; keep raw GBuffer optional |
| AI overfits shader to one screenshot | Regression on other MIs | Validate multiple refs and preserve module/reuse contract |
| Texture import settings differ | Color/roughness mismatch | Include texture import/color-space checks in report |
| Missing CPD/runtime values | Layer/gradient mismatch | Use RenderDoc CPD summary when available; otherwise mark as unresolved |

## Recommended Next Documents

After this design is accepted, create:

```text
Doc/UE_Unity_Material_Semantic_Visual_Validation_Tasks.md
Doc/UE_Unity_Material_Semantic_Visual_Validation_Goal.md
```

The task document should break this into Unity-side tool implementation, Python diff implementation, bundle-doc integration, and fixture validation.
