# UE DXIL Formula And Curve Atlas Evidence Plan

本文档规划两个静态 bundle 分析增强：

```text
1. DXIL Dataflow Formula Summary
   用 selected DXIL entrypoints 验证 Layer / Blend / Function 公式。

2. Curve Atlas Cooked Metadata / Texture Metadata
   从 cooked asset / texture metadata 提取 Curve atlas 行数、尺寸、色彩空间和采样规则。
```

后续约定：

```text
所有 plan / task 文档都输出到 D:\Github\FModel\Doc。
Plan 文档命名使用 *_Plan.md。
Task 文档命名使用 *_Tasks.md。
```

## Scope

目标是提升 Unity Shader 还原时的静态证据质量，让 AI Agent 不必直接吞大量 DXIL 或 cooked JSON。

本计划只覆盖：

```text
DXIL formula evidence
Curve atlas metadata evidence
Unity reconstruction handoff docs update
```

本计划不覆盖：

```text
RenderDoc CPD compact summary
RenderDoc raw export tool changes
Unity-side curvature substitute implementation
Unity shader code generation
```

## Desired Outputs

新增或增强 bundle 输出：

```text
analysis/dxil_formula_evidence.json
analysis/dxil_formula_evidence.md
analysis/module_formula_evidence.json
analysis/curve_atlas_metadata.json
analysis/curve_atlas_metadata.md
source/textures/<CurveAtlasName>.cooked.json
```

Agent docs 需要引用新增证据：

```text
README.md
AGENTS.md
WORKFLOW.md
NEXT_TASK.md
PROMPT_NEXT_SESSION.md
agent_context.json
skills/unity6-urp-deferred-shader-reconstruction/SKILL.md
```

## Phase 1: Curve Atlas Metadata Pass

优先实现 Curve Atlas pass，因为它独立、风险低、产出马上能被 Unity Shader Agent 使用。

### 1.1 Curve Atlas Reference Collector

输入：

```text
parameters/textures.json
analysis/material_layer_parameter_bindings.json
analysis/material_function_dependencies.json
source/material.cooked.json
source/material_functions/*.cooked.json
```

识别规则：

```text
TextureName / ParameterName / ObjectPath 包含：
  Curve
  Gradient
  Atlas
  Curve_Base_Atlas

cooked class / object name 包含：
  UCurveLinearColorAtlas
  CurveLinearColorAtlas
```

输出内部候选：

```json
{
  "Name": "Curve_Base_Atlas",
  "ObjectPath": "/Game/.../Curve_Base_Atlas.0",
  "Source": "analysis/material_layer_parameter_bindings.json",
  "ParameterNames": ["Gradient_Curve", "Curve_Base_Atlas"],
  "Priority": "high"
}
```

### 1.2 Cooked Asset Export

复用已有能力：

```text
SemanticUtilities.ResolvePackage
TextureAssetMetadataExporter pattern
provider.LoadPackage
JsonFiles.Write
```

导出：

```text
source/textures/<CurveAtlasName>.cooked.json
```

支持资产类型：

```text
UCurveLinearColorAtlas
UTexture2D
UVirtualTexture2D
Texture2D fallback
```

如果 cooked package 找不到：

```text
Status = missing
EvidenceLevel = none
Warning 里记录 ObjectPath 和查找失败原因
```

### 1.3 Atlas Fact Extractor

从 cooked JSON 提取：

```text
SizeX
SizeY
PixelFormat
Format
SRGB / bSRGB
CompressionSettings
MipCount
TextureGroup
VirtualTextureStreaming
Curve list
GradientCurves
AtlasHeight
TextureSize
```

如果 cooked metadata 能证明 row count：

```json
{
  "Atlas": "Curve_Base_Atlas",
  "ObjectPath": "/Game/.../Curve_Base_Atlas.0",
  "Size": { "Width": 256, "Height": 16 },
  "ColorSpace": "Linear",
  "RowCount": 16,
  "Rows": [
    {
      "RowIndex": 0,
      "CurveName": "Curve_Foam",
      "V": 0.03125
    }
  ],
  "EvidenceLevel": "cooked_curve_metadata"
}
```

如果只剩 texture metadata：

```json
{
  "Atlas": "Curve_Base_Atlas",
  "Size": { "Width": 256, "Height": 16 },
  "ColorSpace": "Linear",
  "RowCount": null,
  "RowCountCandidates": [8, 16, 32],
  "EvidenceLevel": "texture_dimension_inferred",
  "Warnings": [
    "Curve source metadata not present; row count inferred from texture dimensions and shader sampling evidence is required."
  ]
}
```

### 1.4 Unity Sampling Rule

生成 Unity Agent 可直接消费的采样建议：

```json
{
  "UnitySamplingRule": {
    "U": "input scalar 0..1",
    "V": "(rowIndex + 0.5) / rowCount",
    "ColorSpace": "Linear",
    "RequiresRowCount": true
  }
}
```

如果 row count 不确定：

```text
不要生成 confirmed RowCount。
只生成 RowCountCandidates 和 required follow-up。
```

## Phase 2: DXIL Formula Target Collector

目标是先确定“需要验证哪些模块公式”，不要直接全量解析 DXIL。

输入：

```text
analysis/unity_layer_reconstruction_contract.json
analysis/material_layer_stack.json
analysis/material_layer_parameter_bindings.json
analysis/material_function_dependencies.json
analysis/material_static_permutation.json
analysis/reconstruction_entrypoints.json
analysis/texture_register_candidates.json
analysis/texture_register_statistics.json
analysis/semantic_binding_map.json
analysis/uniform_buffer_usage.json
```

输出内部 target list：

```json
{
  "Schema": "ue-dxil-formula-targets/v1",
  "Targets": [
    {
      "Module": "MB_MaskID",
      "ExpectedFormula": "HeightLerp",
      "RelatedParameters": ["Height", "HeightLerpContrast", "MaskID"],
      "RelatedTextures": ["Mask", "Height"],
      "Priority": "high"
    },
    {
      "Module": "HistogramScan",
      "ExpectedFormula": "position_width_scan",
      "RelatedParameters": ["Position", "Width", "Contrast"],
      "Priority": "high"
    },
    {
      "Module": "ML_LayerTint",
      "ExpectedFormula": "CheapContrastOrTintMask",
      "RelatedParameters": ["Tint", "Contrast", "Mask"],
      "Priority": "medium"
    }
  ]
}
```

Selection rule：

```text
只读 analysis/reconstruction_entrypoints.json 选出的 PrimaryPixelShaders 和 SupportingShaders。
不要扫描所有 shaders/*.dxil.ll。
```

## Phase 3: DXIL Dataflow Reader

在现有 DXIL LL parser / semantic analyzer 基础上扩展，不做完整 HLSL 反编译器。

### 3.1 Instruction Model

需要解析或复用的 DXIL 证据：

```text
texture sample
cbuffer load
resource load
fadd / fsub / fmul / fdiv
fmuladd / mad-like chains
min / max
clamp / saturate pattern
dot / normalize
select / cmp
output store / SV_Target
```

记录：

```json
{
  "Shader": "shaders/023_..._group_04265.dxil.ll",
  "InstructionId": "%123",
  "Op": "fmul",
  "Inputs": ["%121", "%122"],
  "Register": "cb0",
  "Offset": 384,
  "Channels": ["x"]
}
```

### 3.2 SSA Slicer

切片方向：

```text
从 GBuffer / SV_Target output 向前追踪。
从 texture sample result 向后追踪到 blend alpha / color / normal / roughness。
从 cbuffer load 向后追踪到 arithmetic chain。
```

输出紧凑 slice：

```json
{
  "SliceId": "shader023_target0_alpha_chain",
  "Outputs": ["SV_Target0.a"],
  "Inputs": [
    { "Kind": "TextureSample", "Register": "t3", "Channels": ["r"] },
    { "Kind": "CBufferLoad", "Register": "cb0", "Offset": 384 }
  ],
  "Operations": ["fsub", "fmul", "saturate", "lerp"]
}
```

## Phase 4: Pattern Recognizers

第一版只做有限公式识别。

### 4.1 HeightLerp

识别模式：

```text
heightA / heightB / bias / contrast
saturate
lerp(A, B, alpha)
```

输出：

```json
{
  "FormulaCandidate": "HeightLerp",
  "EvidenceLevel": "dxil_dataflow_supported",
  "Pattern": "saturate((heightA - heightB + bias) * contrast)",
  "Inputs": [
    { "Kind": "TextureSample", "Register": "t3", "Channels": ["r"] },
    { "Kind": "CBufferLoad", "Register": "cb0", "Offset": 384 }
  ],
  "Operations": ["fsub", "fadd", "fmul", "saturate", "lerp"]
}
```

### 4.2 CheapContrast

识别模式：

```text
(x - 0.5) * contrast + 0.5
saturate variant
```

输出必须标注作用对象：

```text
mask
height
tint
roughness
unknown
```

### 4.3 HistogramScan

识别模式：

```text
abs(x - position)
width divisor / multiplier
smoothstep-like
saturate band mask
```

重点输出：

```text
Position parameter candidate
Width parameter candidate
Contrast / falloff candidate
mask output chain
```

### 4.4 Mask Channel Selection

识别：

```text
sample.r/g/b/a -> blend alpha
```

输出：

```json
{
  "TextureRegister": "t4",
  "Channel": "g",
  "Use": "blend_alpha_candidate",
  "EvidenceLevel": "dxil_dataflow_supported"
}
```

### 4.5 Normal Intensity / Normal Blend

识别：

```text
normal unpack
xy scale
z recompute / normalize
blend / overlay normal
```

### 4.6 Curve Atlas Sampling

识别：

```text
sample atlas texture
u from input scalar / mask
v from rowIndex / rowCount
v = (row + 0.5) / N
```

如果发现 divisor：

```text
把 N 回传给 curve_atlas_metadata.json 的 RowCountEvidence。
```

## Phase 5: Module Formula Evidence Aggregator

把 DXIL pattern 映射回模块。

输出：

```text
analysis/module_formula_evidence.json
```

Schema：

```json
{
  "Schema": "ue-module-formula-evidence/v1",
  "Material": "/Game/...",
  "GeneratedAtUtc": "...",
  "Modules": [
    {
      "Module": "MB_MaskID",
      "Status": "supported",
      "Confidence": 0.82,
      "ImplementedFormulaRecommendation": "Use HeightLerp branch",
      "Evidence": [
        {
          "Shader": "shaders/023_..._group_04265.dxil.ll",
          "Pattern": "HeightLerp",
          "EvidenceLevel": "dxil_dataflow_supported",
          "Inputs": [
            { "Kind": "TextureSample", "Register": "t3", "Channels": ["r"] },
            { "Kind": "CBufferLoad", "Register": "cb0", "Offset": 384 }
          ]
        }
      ],
      "Unresolved": [
        "Could not map cb0 offset 384 to parameter name."
      ]
    }
  ],
  "Warnings": [
    "Anonymous t?/cb? bindings remain cooked shader evidence, not UE source graph node names."
  ]
}
```

Status values：

```text
supported
partial
not_found
conflict
not_applicable
```

## Phase 6: Markdown Summaries

Generate AI-friendly summaries:

```text
analysis/dxil_formula_evidence.md
analysis/curve_atlas_metadata.md
```

Rules:

```text
1. Keep summaries compact.
2. Link to JSON files and selected shader paths.
3. Do not paste large DXIL blocks.
4. Separate confirmed facts from inferred assumptions.
5. State unresolved anonymous register/cbuffer mappings explicitly.
```

## Phase 7: Agent Handoff Update

Update generated bundle docs to add these files to read-first list:

```text
analysis/module_formula_evidence.json
analysis/dxil_formula_evidence.json
analysis/curve_atlas_metadata.json
analysis/curve_atlas_metadata.md
```

Update generated instructions:

```text
Before implementing Layer / Blend / Function formulas, read module_formula_evidence.json.
Before implementing curve gradient / curve atlas sampling, read curve_atlas_metadata.json.
Do not claim source graph equivalence when evidence is only DXIL dataflow.
```

## Validation Plan

Use `MI_CG_RockSmooth_01a` as first fixture.

Expected high-value checks:

```text
1. analysis/curve_atlas_metadata.json exists and is parseable.
2. Curve_Base_Atlas is detected if referenced by the bundle.
3. RowCount is either confirmed with EvidenceLevel or left as candidates with warning.
4. analysis/module_formula_evidence.json exists and is parseable.
5. MB_MaskID reports supported / partial / not_found, not silent missing.
6. HistogramScan reports supported / partial / not_found, not silent missing.
7. CheapContrast usage reports supported / partial / not_found.
8. DXIL evidence references selected shaders only.
9. Raw shaders/*.dxil.ll are not inlined into markdown.
10. Bundle verifier still passes.
```

Regression commands:

```powershell
dotnet build CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release

dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --semantic-only "BUNDLE_DIR" `
  --verbose

dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --verify-only "BUNDLE_DIR" `
  --verbose
```

## Acceptance Criteria

```text
1. Curve Atlas pass can locate and describe curve atlas assets without RenderDoc.
2. Curve atlas metadata records ObjectPath, cooked JSON path, size, color space, and row count evidence.
3. DXIL formula pass reads selected entrypoints only.
4. DXIL formula pass emits formula evidence for HeightLerp, CheapContrast, HistogramScan, mask channel selection, normal intensity/blend, and curve atlas sampling when patterns are present.
5. module_formula_evidence.json maps formula evidence back to Master / Layer / Blend / Function modules.
6. Evidence levels distinguish confirmed cooked dataflow from inferred source intent.
7. Generated Agent docs tell Unity Shader Agent to read module_formula_evidence and curve_atlas_metadata before raw DXIL.
8. Existing bundle export, semantic-only, context-only, and verify-only workflows keep working.
```

## Implementation Notes

Do not overbuild a generic decompiler in the first version.

Preferred first implementation:

```text
1. Build curve atlas metadata pass.
2. Build formula target collector.
3. Extend existing DXIL LL parser just enough for selected pattern recognition.
4. Emit conservative evidence with status partial/not_found when uncertain.
5. Update Agent docs and verifier expectations after outputs stabilize.
```

