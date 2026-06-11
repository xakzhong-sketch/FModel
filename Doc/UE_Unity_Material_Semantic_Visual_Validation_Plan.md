# UE Unity Material Semantic Visual Validation Plan

本文档基于 `UE_Unity_Material_Semantic_Visual_Validation_Design.md`，定义可落地的实现计划。

目标是在材质属性恢复完成后，为 Unity 6 URP Deferred shader 还原增加一个稳定的视觉验证闭环：

```text
Restore .mat properties -> Unity semantic capture -> diff/report -> AI shader fix -> repeat
```

## Scope

实现范围：

- Unity 侧材质语义截图工具。
- FModel/Python 侧 diff/report 工具。
- 新的 `/goal` 文档。
- Bundle Agent 文档集成。
- 视觉验证工作目录规范。

不在本计划内：

- 自动从 UE 源材质图恢复节点。
- 在 RenderDoc/qrenderdoc 工具中加入 Unity 视觉验证。
- 把 UE Deferred LightPass 移植进 Unity material shader。
- 将最终 lit screenshot 作为唯一还原度依据。

## Guiding Rules

1. 语义图优先，最终画面辅助。
2. Unity 和 UE raw GBuffer 不做直接一一对应比较，必须先解码到 semantic level。
3. Shader debug pass 必须复用真实 shader module/surface evaluation，避免另写一套逻辑。
4. Diff/report 必须 compact，AI Agent 默认读 report/contact sheet，不读全量截图。
5. 材质属性恢复和 shader 修正分阶段：
   - `UE_Unity_Material_Property_Restore_Goal.md` 只负责 `.mat` 属性。
   - 新视觉验证 Goal 负责 capture/diff/report。
   - shader 修正仍回到 bundle reconstruction/fix loop。
6. Unity target 保持 Unity 6 URP Deferred + DOTS instancing。

## Target Workflow

用户操作顺序：

```text
1. /goal D:\Github\FModel\Doc\UE_Cooked_Material_Bundle_Export_Goal.md 导出 MI_xxx 材质
2. /goal MI_xxx.bundle\PROMPT_NEXT_SESSION.md
3. /goal D:\Github\FModel\Doc\UE_Unity_Material_Property_Restore_Goal.md Apply Mat=... Bundle=...
4. 在工作目录放入 VisualRefs/*.png + optional masks/meta
5. /goal D:\Github\FModel\Doc\UE_Unity_Material_Semantic_Visual_Validation_Goal.md Mat=... Bundle=...
6. 如果 report fail，AI Agent 修改 shader/module 后重新执行第 5 步
```

简化目录模式：

```text
cd K:\WorkSpace\ShaderReverse\MI_CG_RockSmooth_01a
/goal D:\Github\FModel\Doc\UE_Unity_Material_Semantic_Visual_Validation_Goal.md
```

自动检测：

- 当前目录下唯一 `.bundle`。
- 当前目录下 `VisualRefs`。
- 当前目录下 `UnityValidation/config.json`，若不存在则生成。
- `Mat=...` 若未提供，尝试从 bundle/material restore context 推断；多候选时要求显式传入。

## Output Layout

```text
<MaterialWorkDir>/
  VisualRefs/
  UnityValidation/
    config.json
    unity_capture.log
    captures/
      capture_manifest.json
      base_color.png
      normal_ws.exr
      roughness.png
      metallic.png
      ao.png
      emission.png
      alpha.png
      final_color.png
    reports/
      visual_validation_report.json
      visual_validation_report.md
      visual_validation_advice.json
      contact_sheet.png
      diff_base_color.png
      diff_normal_ws.png
      diff_roughness.png
```

If bundle exists, optionally mirror compact validation report into:

```text
<Bundle>/analysis/unity_visual_validation/
  visual_validation_report.json
  visual_validation_report.md
  visual_validation_advice.json
```

Do not copy large captures into bundle analysis by default.

## Phase 1 - Shader Semantic Debug Contract

### Files

Unity project:

```text
Assets/Shaders/Subnautica2/Debug/SN2SemanticDebug.hlsl
```

Generated/reconstructed shader modules:

```text
Assets/Shaders/Subnautica2/Reconstructed/*.shader
Assets/Shaders/Subnautica2/MaterialModules/*.hlsl
```

### Implementation

Define a shared semantic surface structure, for example:

```hlsl
struct SN2DebugSurface
{
    float3 BaseColor;
    float3 NormalWS;
    float3 NormalTS;
    float Roughness;
    float Smoothness;
    float Metallic;
    float AmbientOcclusion;
    float3 Emission;
    float Alpha;
    float OpacityMask;
    float4 LayerBlend;
    float4 HeightBlend;
};
```

Add output helpers:

```hlsl
float4 SN2EncodeDebugBaseColor(SN2DebugSurface s);
float4 SN2EncodeDebugNormalWS(SN2DebugSurface s);
float4 SN2EncodeDebugRoughness(SN2DebugSurface s);
...
```

Add fixed semantic debug passes first, with material/global selector as fallback only:

```text
SN2SemanticDebug_BaseColor
SN2SemanticDebug_NormalWS
SN2SemanticDebug_NormalTS
SN2SemanticDebug_Roughness
SN2SemanticDebug_Smoothness
SN2SemanticDebug_Metallic
SN2SemanticDebug_AmbientOcclusion
SN2SemanticDebug_Emission
SN2SemanticDebug_Alpha
SN2SemanticDebug_OpacityMask
SN2SemanticDebug_LayerBlend
SN2SemanticDebug_HeightBlend
SN2SemanticDebug_FinalColor
```

The runner should prefer fixed pass names. A single `SN2SemanticDebug` pass driven by `_SN2DebugMode` is allowed for backward compatibility, but fixed passes are more deterministic in Unity batch capture.

### Acceptance

- A reconstructed shader can render at least BaseColor, NormalWS, Roughness, Metallic, AO, Alpha debug outputs.
- Debug pass calls the same material/layer evaluation function as `UniversalGBuffer`.
- Debug code does not remove or break DOTS instancing macros.

## Phase 2 - Shader Generator / Agent Contract Update

### Files

FModel side:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Tools/generate_unity_layer_stack_shader.py
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/AgentWorkspaceWriter.cs
```

Generated bundle docs:

```text
AGENTS.md
WORKFLOW.md
PROMPT_NEXT_SESSION.md
skills/unity6-urp-deferred-shader-reconstruction/SKILL.md
```

### Implementation

- Add instruction that high-fidelity reconstructed shaders should expose semantic debug outputs.
- Update shader generator only if it is still used to produce shader skeletons.
- Generated shader/module code should share material evaluation between:
  - `UniversalGBuffer`
  - debug semantic pass
  - optional preview/final forward pass if present

### Acceptance

- New bundle docs tell Agent to include semantic debug support when writing Unity shaders.
- Existing shader reconstruction path still targets Unity 6 URP Deferred + DOTS.
- No Unity `.mat` work is added to `PROMPT_NEXT_SESSION.md`.

## Phase 3 - Unity Capture Runtime/Editor Tool

### Files

Unity project:

```text
Assets/ShaderReverse/Runtime/Validation/URPMaterialSemanticCaptureFeature.cs
Assets/Editor/ShaderReverse/Validation/SN2MaterialVisualValidationRunner.cs
Assets/ShaderReverse/Validation/Scenes/SN2MaterialPreview.unity
```

Optional template in FModel:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Tools/unity_shader_validation/SN2MaterialVisualValidationRunner.cs.txt
CUE4Parse/CUE4Parse.ShaderBundleExporter/Tools/unity_shader_validation/URPMaterialSemanticCaptureFeature.cs.txt
```

### Implementation

Editor runner:

1. Parse `UnityValidation/config.json`.
2. Open preview scene.
3. Load target `.mat`.
4. Assign it to preview mesh.
5. Configure camera, resolution, lighting, postprocess off for semantic capture.
6. Render selected capture modes.
7. Save PNG/EXR outputs.
8. Write `capture_manifest.json`.

Capture modes:

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
LayerBlend
HeightBlend
FinalColor
```

Optional raw/decoded GBuffer modes:

```text
RawGBuffer0
RawGBuffer1
RawGBuffer2
RawGBuffer3
DecodedGBufferBaseColor
DecodedGBufferNormalWS
DecodedGBufferRoughness
DecodedGBufferMetallic
Depth
```

### Command

```powershell
Unity.exe -batchmode -projectPath "UNITY_PROJECT" `
  -executeMethod SN2MaterialVisualValidationRunner.Run `
  -validationConfig "WORK_DIR\UnityValidation\config.json" `
  -logFile "WORK_DIR\UnityValidation\unity_capture.log"
```

Do not add `-nographics`. Unity RenderTexture/semantic capture needs a graphics device; testing with `-nographics` produced undefined gray captures.

### Acceptance

- Command runs without UI interaction.
- Captures are written with deterministic names.
- Manifest records Unity version, URP, Deferred, material path, shader path, scene, camera, resolution, and capture modes.
- Missing shader debug pass produces a clear error in manifest/report.

## Phase 4 - Python Diff And Report Tool

### Files

FModel side:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Tools/unity_visual_validation_diff.py
CUE4Parse/CUE4Parse.ShaderBundleExporter/Tools/unity_visual_validation_advisor.py
```

### Inputs

```text
--config WORK_DIR\UnityValidation\config.json
--captures WORK_DIR\UnityValidation\captures
--refs WORK_DIR\VisualRefs
--bundle WORK_DIR\MI_xxx.bundle
--out WORK_DIR\UnityValidation\reports
```

### Implementation

Diff script:

- Load config.
- Load capture manifest.
- Load reference metadata and masks.
- Match semantics:
  - Unity semantic capture vs UE/RenderDoc semantic reference if available.
  - Unity final color vs masked original screenshot as secondary evidence.
- Compute compact metrics.
- Write report JSON/Markdown.
- Generate contact sheet and top diff images.

Metrics:

```text
MAE
RMSE
masked MAE
normal angle error
histogram delta
roughness mean delta
metallic mean delta
```

Advisor:

- Classify likely issue:

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

### Acceptance

- All report JSON files parse.
- Markdown report starts with pass/fail summary and top failing semantics.
- Contact sheet is generated when image dependencies are available.
- Report recommends exact next files/modules to inspect when possible.
- Large screenshots are not embedded into Markdown.

## Phase 5 - Goal Document

### File

```text
Doc/UE_Unity_Material_Semantic_Visual_Validation_Goal.md
```

### Goal Behavior

The goal should:

1. Resolve workspace root.
2. Resolve bundle path.
3. Resolve Unity `.mat` path.
4. Validate `VisualRefs` existence.
5. Generate/update `UnityValidation/config.json`.
6. Install or verify Unity validation scripts if missing.
7. Run Unity capture command.
8. Run Python diff/advisor.
9. Optionally mirror compact reports into bundle analysis.
10. If fail, give AI Agent a scoped fix loop.

### Invocation

```text
/goal D:\Github\FModel\Doc\UE_Unity_Material_Semantic_Visual_Validation_Goal.md

/goal D:\Github\FModel\Doc\UE_Unity_Material_Semantic_Visual_Validation_Goal.md Mat=... Bundle=...

/goal D:\Github\FModel\Doc\UE_Unity_Material_Semantic_Visual_Validation_Goal.md Root=K:\WorkSpace\ShaderReverse\MI_CG_RockSmooth_01a
```

### Acceptance

- Goal can run from material work directory.
- Goal does not scan all Unity `Assets`.
- Goal does not modify `.mat` unless explicitly delegated to material restore Goal.
- Goal outputs final status, report paths, and top issues.

## Phase 6 - Bundle Agent Docs Integration

### Files

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/AgentWorkspaceWriter.cs
```

Generated docs:

```text
README.md
AGENTS.md
WORKFLOW.md
NEXT_TASK.md
PROMPT_NEXT_SESSION.md
agent_context.json
skills/unity6-urp-deferred-shader-reconstruction/SKILL.md
```

### Implementation

Add optional visual validation context:

```json
{
  "UnityVisualValidation": {
    "Exists": true,
    "Directory": "UnityValidation/reports",
    "ReportMarkdown": "UnityValidation/reports/visual_validation_report.md",
    "ReportJson": "UnityValidation/reports/visual_validation_report.json",
    "AdviceJson": "UnityValidation/reports/visual_validation_advice.json"
  }
}
```

Docs should say:

- After material restore Apply, run semantic visual validation if `VisualRefs` exists.
- Use `visual_validation_report.md/json` before raw screenshots.
- Do not use final lit screenshot alone as proof.
- If validation fails, fix shader/module logic or material property binding according to issue classification.

### Acceptance

- `--context-only` refresh detects compact visual validation reports if mirrored into bundle analysis.
- Bundle read-first includes visual validation report only after cooked/static and RenderDoc compact evidence.

## Phase 7 - Optional URP Raw GBuffer Capture

### Trigger

Implement only after semantic debug capture is working.

### Implementation

Add RenderGraph/ScriptableRenderPass capture after URP GBuffer and before deferred lighting.

Outputs:

```text
raw_gbuffer0.exr
raw_gbuffer1.exr
raw_gbuffer2.exr
raw_gbuffer3.exr
decoded_gbuffer_base_color.png
decoded_gbuffer_normal_ws.exr
decoded_gbuffer_roughness.png
decoded_gbuffer_metallic.png
depth.exr
```

### Acceptance

- Raw capture does not replace semantic debug capture.
- Report labels raw URP GBuffer as Unity pipeline evidence, not source shader semantic proof.
- Decode contract records Unity version and URP pipeline settings.

## Phase 8 - Fixtures And Verification

### Fixture 1 - MI_CG_RockSmooth_01a

Inputs:

```text
Bundle: D:\Tmp\ShaderBundles\MI_CG_RockSmooth_01a.bundle or workspace bundle
Mat: K:\WorkSpace\trunk\ExportedProject\Assets\Art\Environment\Biome\CoralGarden\Rocks\Material\MI_CG_RockSmooth_01a.mat
VisualRefs: user-provided screenshots/masks
RenderDoc: optional PS GBuffer compact summary
```

Expected:

- Captures generated.
- BaseColor, NormalWS, Roughness, Metallic, AO reports generated.
- If mismatch exists, advisor classifies likely issue.

### Fixture 2 - Simple Non-Layer Material

Purpose:

- Verify minimal shader path.
- Validate no layer-specific assumptions.

### Fixture 3 - Missing Texture Case

Purpose:

- Confirm validation reports `texture_missing_or_wrong_guid`.
- Confirm it does not silently rewrite `.mat`.

## Completion Criteria

The feature is complete when:

1. Unity semantic capture can run in batchmode for one restored material.
2. Diff/report tool generates parseable JSON and readable Markdown.
3. Report identifies at least BaseColor, NormalWS, Roughness, Metallic, AO status.
4. Visual validation Goal can be invoked from a material work directory.
5. Bundle docs explain the post-restore validation workflow.
6. AI Agent can use report-first workflow without reading all screenshots.
7. Remaining differences are classified as shader logic, material binding, texture import, runtime feature, lighting/postprocess, or reference limitation.
8. Existing bundle export, RenderDoc compact summary, and material restore workflows remain unchanged unless this validation goal is explicitly invoked.

## Risk Management

| Risk | Mitigation |
|---|---|
| Debug pass diverges from real GBuffer pass | Share module evaluation function |
| Reference screenshot not comparable | Require mask/meta and classify as reference limitation |
| Agent overfits one screenshot | Support multiple refs and semantic thresholds |
| Unity raw GBuffer decode changes by version | Keep raw GBuffer optional and version-stamped |
| Shader lacks debug support | Report explicit missing debug pass instead of failing silently |
| Texture import settings cause false mismatch | Include texture color-space/import checks in advisor |
| Batch Unity path varies | Config stores Unity project; Goal allows explicit UnityExe/UnityRoot |

## Deliverables

Plan deliverables:

```text
Doc/UE_Unity_Material_Semantic_Visual_Validation_Plan.md
Doc/UE_Unity_Material_Semantic_Visual_Validation_Tasks.md
Doc/UE_Unity_Material_Semantic_Visual_Validation_Goal.md
```

Implementation deliverables:

```text
Assets/Shaders/Subnautica2/Debug/SN2SemanticDebug.hlsl
Assets/Editor/ShaderReverse/Validation/SN2MaterialVisualValidationRunner.cs
Assets/ShaderReverse/Runtime/Validation/URPMaterialSemanticCaptureFeature.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Tools/unity_visual_validation_diff.py
CUE4Parse/CUE4Parse.ShaderBundleExporter/Tools/unity_visual_validation_advisor.py
```

Generated runtime outputs:

```text
UnityValidation/config.json
UnityValidation/captures/capture_manifest.json
UnityValidation/reports/visual_validation_report.json
UnityValidation/reports/visual_validation_report.md
UnityValidation/reports/visual_validation_advice.json
UnityValidation/reports/contact_sheet.png
```

## Recommended Next Step

Create:

```text
Doc/UE_Unity_Material_Semantic_Visual_Validation_Tasks.md
```

Then implement in this order:

1. Python diff/report scaffold with synthetic image fixture.
2. Unity batch capture runner template.
3. Shader semantic debug include/pass contract.
4. Goal document and Agent workflow.
5. Bundle docs integration.
6. Optional raw URP GBuffer capture.
