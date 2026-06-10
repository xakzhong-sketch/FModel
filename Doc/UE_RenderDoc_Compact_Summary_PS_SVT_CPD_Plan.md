# UE RenderDoc Compact Summary PS/SVT/CPD Enhancement Plan

本文档定义下一阶段对 `CUE4Parse.ShaderBundleExporter/Tools/renderdoc_compact_summary.py` 的增强计划。

目标是把非 Nanite `DrawIndexedInstanced` 的 RenderDoc current-drawcall 导出，压缩成更适合 Unity Shader 还原 Agent 使用的 runtime evidence，尤其覆盖：

- Pixel Shader GBuffer/BasePass drawcall。
- Streaming Virtual Texturing runtime 资源识别。
- Custom Primitive Data 运行时链路和值提取。
- PS/VS IO、texture/cbuffer/UAV、MRT/GBuffer 输出格式。
- 与 cooked UE shader bundle 的静态证据合并。

## Background

当前 compact summary 已能输出：

```text
renderdoc_runtime_overlay.json
renderdoc_runtime_overlay.md
renderdoc_shader_match.json
renderdoc_texture_slot_map.json
renderdoc_sampler_slot_map.json
renderdoc_cbuffer_value_usage.json
renderdoc_runtime_outputs.json
renderdoc_drawcall_role.json
interesting_dxil_snippets.md
agent_notes.md
```

但当前实现仍偏“资源清单/overlay”，对高保真 Unity shader 还原还有几个缺口：

- 不能明确标出 SVT page table / physical texture / feedback UAV。
- 不能从 VS/PS 链路里总结 `CUSTOM_DATA_OFFSET` / `CUSTOM_DATA_COUNT`。
- 不能从 `StructuredBuffer0` raw data 中抽取 CPD float4 值。
- 不能把 PS `SV_Target0..3` 与 runtime MRT 格式整理成一个 GBuffer summary。
- DXIL snippets 还没有针对 CPD、SVT、GBuffer 输出做专门选择。
- `renderdoc_cbuffer_value_usage.json` 仍是 metadata-only，对 Agent 帮助有限。

## Guiding Rules

1. RenderDoc 是运行时证据，不是 UE 源材质图。
2. 不把 raw buffer、大贴图、完整 DXIL 全量塞进 compact context。
3. SVT runtime 不移植到 Unity shader；Unity 还原应转为直接采样 Unity 贴图属性。
4. CPD 值可以作为运行时事实，但 CPD 字段语义仍需要结合 material layer/function/cooked evidence。
5. CPD summary 只能有一套实现和一套 Agent-facing 输出；qrenderdoc 菜单、CLI、FModel goal 都应调用同一套逻辑。
6. 所有结论必须区分：
   - `runtime_observed`
   - `dxil_dataflow_supported`
   - `name_or_format_inferred`
   - `unresolved`

## Target Input Example

本计划以这类 RenderDoc current-drawcall export 为目标：

```text
RenderDocCapture/
  EID_6250_ID3D12GraphicsCommandList_DrawIndexedInstanced/
    drawcall.json
    pipeline_state.json
    shader_reconstruction_index.json
    Shaders/ps/reflection.json
    Shaders/ps/disassembly_native_DXBC_DXIL.txt
    Shaders/ps/raw_shader.bin
    Shaders/vs/reflection.json
    Shaders/vs/disassembly_native_DXBC_DXIL.txt
    Textures/Metadata/textures.json
    Textures/Metadata/samplers.json
    Textures/Inputs/*
    Textures/Outputs/*
    ConstantBuffers/constant_buffers.json
    ConstantBuffers/ps/*.variables.csv
    ResourceBuffers/resource_buffers.json
    ResourceBuffers/*.raw.bin
    Mesh/vertex_input_layout.json
    Mesh/mesh_postvs.json
```

The implementation must also continue to support older D3D11/D3D12 captures and Nanite/CS captures as partial output.

## Responsibility Boundary

RenderDoc/qrenderdoc 只负责导出 drawcall 原始数据，不负责运行 CPD summary，不生成 CPD 分析结论。

RenderDoc 导出工具的职责到这里为止：

```text
drawcall.json
pipeline_state.json
shader_reconstruction_index.json
Shaders/*/reflection.json
Shaders/*/raw_shader.bin
Shaders/*/disassembly_native_*.txt
Textures/Metadata/textures.json
Textures/Metadata/samplers.json
Textures/Inputs/*
Textures/Outputs/*
ConstantBuffers/constant_buffers.json
ConstantBuffers/*/*.variables.csv
ConstantBuffers/*/*.raw.bin
ResourceBuffers/resource_buffers.json
ResourceBuffers/*.raw.bin
Mesh/vertex_input_layout.json
Mesh/mesh_postvs.json
Mesh/*.bin
```

FModel compact summary 工具负责从这些原始导出数据中生成 CPD/SVT/GBuffer 等 compact evidence。

最终规则：

```text
RenderDoc/qrenderdoc:
  raw export only

FModel renderdoc_compact_summary.py:
  CPD summary
  SVT summary
  GBuffer summary
  shader IO summary
  material runtime evidence summary
```

不要在 qrenderdoc 里增加 `Tools -> Run CPD Summary Analyzer...` 这类分析入口。后续如果需要一个独立 CLI，也应放在 FModel 侧，例如：

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Tools/renderdoc_compact_summary.py
```

或者作为 FModel 侧内部模块：

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Tools/shader_reconstruction/cpd_summary.py
```

但它仍然属于 FModel workflow，不属于 RenderDoc 导出工具。

### CPD Analyzer Ownership

FModel-side CPD analyzer owns:

- export index parsing
- resource candidate scoring
- DXBC/DXIL load parser
- resolver
- raw buffer window extraction
- UE GPUScene layout interpretation
- bundle hints
- finding scoring
- Markdown/JSON report writer

`renderdoc_compact_summary.py` owns orchestration:

- running the CPD analyzer pass
- merging CPD facts with shader IO, SVT, GBuffer, and cooked bundle evidence
- writing compact read-order guidance
- preventing raw context bloat

### Stable CPD Output

Recommended stable fields:

```json
{
  "Schema": "ue-renderdoc-cpd-summary/v1",
  "Producer": {
    "Tool": "renderdoc_compact_summary.py",
    "Pass": "cpd_summary",
    "AnalyzerModule": "renderdoc_compact_summary.cpd_summary"
  },
  "Status": "success|partial|not_found",
  "Detected": true,
  "InputSemantics": {},
  "VsToPsPath": {},
  "RuntimeBuffers": [],
  "AccessPatterns": [],
  "SampledValues": [],
  "Layout": {
    "Name": "UE5_6_GPUScene",
    "Source": "UE5_6_GPUScene.example.json|custom|unknown"
  },
  "Findings": [],
  "Limitations": []
}
```

## Output Additions

Keep existing files and add the following compact outputs:

```text
renderdoc_shader_io_summary.json
renderdoc_shader_io_summary.md
renderdoc_svt_summary.json
renderdoc_svt_summary.md
renderdoc_cpd_summary.json
renderdoc_cpd_summary.md
renderdoc_gbuffer_outputs.json
renderdoc_gbuffer_outputs.md
renderdoc_material_runtime_evidence.json
renderdoc_material_runtime_evidence.md
```

`renderdoc_runtime_overlay.json` should reference these files in `RecommendedReadOrder`.

If a bundle is provided, outputs still go to:

```text
<Bundle>/analysis/renderdoc/
```

If no bundle is provided, outputs go to:

```text
<RenderDocDrawcallDir>/CompactSummary/
```

## Output Schemas

### `renderdoc_shader_io_summary.json`

Purpose: compact PS/VS input/output signature and stage role evidence.

Required fields:

```json
{
  "Schema": "ue-renderdoc-shader-io-summary/v1",
  "Stages": [
    {
      "Stage": "ps",
      "EntryPoint": "MainPS",
      "Encoding": "DXIL",
      "Inputs": [
        {
          "Semantic": "CUSTOM_DATA_OFFSET",
          "Register": "4.y",
          "Type": "uint",
          "Interpolation": "nointerpolation",
          "EvidenceLevel": "runtime_reflection"
        }
      ],
      "Outputs": [
        {
          "Semantic": "SV_Target0",
          "Register": "0",
          "Type": "float4",
          "RuntimeTarget": "rt0"
        }
      ],
      "SourceFiles": []
    }
  ]
}
```

### `renderdoc_svt_summary.json`

Purpose: classify Streaming Virtual Texturing runtime resources.

Required fields:

```json
{
  "Schema": "ue-renderdoc-svt-summary/v1",
  "Detected": true,
  "EvidenceLevel": "runtime_binding_and_format_supported",
  "PageTableTextures": [
    {
      "Stage": "ps",
      "Register": "t3",
      "Format": "R32G32_UINT",
      "Width": 384,
      "Height": 256,
      "Resource": "ResourceId::336229"
    }
  ],
  "PhysicalTextureCandidates": [
    {
      "Stage": "ps",
      "Register": "t10",
      "Format": "BC7_TYPELESS",
      "Width": 5712,
      "Height": 5712,
      "Resource": "ResourceId::10660"
    }
  ],
  "FeedbackOrRuntimeUavs": [
    {
      "Stage": "ps",
      "Register": "u0",
      "ResourceName": "RWStructuredBuffer0"
    }
  ],
  "UnityHandling": "Do not port SVT indirection; reconstruct Unity shader with direct texture properties and mark VT runtime as abstracted.",
  "KnownLimitations": []
}
```

Classification heuristics:

- Page table candidate:
  - Texture input format `R32G32_UINT`, `R32_UINT`, or similar integer format.
  - Resource name is render target or shader-read/color-target page resource.
  - Low/medium dimension such as 256/384/512 and multiple repeated registers.
- Physical texture candidate:
  - Compressed format such as `BC1`, `BC3`, `BC5`, `BC7`.
  - Large dimensions.
  - Multiple registers can point to same resource id.
- Feedback/UAV candidate:
  - `RWStructuredBuffer`, `RWTexture`, `u#`.
  - Shader snippet includes `Interlocked`, `Store`, or feedback-like writes.

### `renderdoc_cpd_summary.json`

Purpose: extract Custom Primitive Data runtime evidence and values.

Required fields:

```json
{
  "Schema": "ue-renderdoc-cpd-summary/v1",
  "Status": "success|partial|not_found",
  "Detected": true,
  "InputSemantics": {
    "PrimitiveId": "PRIMITIVE_ID",
    "CustomDataOffset": "CUSTOM_DATA_OFFSET",
    "CustomDataCount": "CUSTOM_DATA_COUNT"
  },
  "VsToPsPath": {
    "VsOutputsCustomData": true,
    "PsConsumesCustomData": true,
    "Evidence": []
  },
  "RuntimeBuffers": [
    {
      "Stage": "ps",
      "Register": "t0",
      "ResourceName": "StructuredBuffer0",
      "StrideBytes": 16,
      "RawPath": "ResourceBuffers/...",
      "UsedAs": "custom_primitive_data_values_candidate"
    }
  ],
  "AccessPatterns": [
    {
      "Register": "t0",
      "IndexExpression": "CUSTOM_DATA_OFFSET + 0",
      "Condition": "CUSTOM_DATA_COUNT > 0",
      "Components": ["x", "y", "z", "w"],
      "EvidenceLines": []
    }
  ],
  "SampledValues": [
    {
      "Index": 0,
      "Source": "StructuredBuffer0[CUSTOM_DATA_OFFSET + 0]",
      "Float4": [0.0, 0.0, 0.0, 0.0],
      "EvidenceLevel": "runtime_buffer_decoded"
    }
  ],
  "Limitations": []
}
```

Implementation detail:

- Parse VS reflection for output semantics:
  - `PRIMITIVE_ID`
  - `CUSTOM_DATA_OFFSET`
  - `CUSTOM_DATA_COUNT`
- Parse PS reflection for matching input semantics.
- Parse PS native DXIL/disassembly for:
  - `_IN.CUSTOM_DATA_OFFSET`
  - `_IN.CUSTOM_DATA_COUNT`
  - `StructuredBuffer0.Load(_206)`
  - `StructuredBuffer0.Load(_206 + N)`
- Resolve `StructuredBuffer0` to `ResourceBuffers/resource_buffers.json`.
- Decode raw buffer as `float4` when reflection/disassembly says `StructuredBuffer<float4>` / stride 16.
- Default sample budget:
  - Decode only the observed base offset and `base + 0..ceil(count/4)` if the offset can be determined.
  - If actual per-instance offset cannot be determined, decode no raw values and output only access patterns.

Important limitation:

If post-VS export does not expose actual `CUSTOM_DATA_OFFSET` values per vertex/instance, the summary can prove the CPD path but may not know the exact runtime offset. In that case:

```json
"Status": "partial",
"Reason": "CPD path is proven, but actual per-instance CUSTOM_DATA_OFFSET value was not extracted."
```

### `renderdoc_gbuffer_outputs.json`

Purpose: summarize PS outputs and MRT/GBuffer target formats.

Required fields:

```json
{
  "Schema": "ue-renderdoc-gbuffer-outputs/v1",
  "DrawcallRole": "BasePassGBuffer",
  "Targets": [
    {
      "Slot": "rt0",
      "Semantic": "SV_Target0",
      "Format": "R16G16B16A16_FLOAT",
      "Width": 1920,
      "Height": 1080,
      "CandidateMeaning": "scene_color_or_gbuffer0_candidate",
      "EvidenceLevel": "runtime_output_format_supported"
    }
  ],
  "Depth": {},
  "UnityHandling": "Map material surface outputs into Unity URP Deferred GBuffer-compatible shader; reuse URP deferred lighting."
}
```

Role classifier improvement:

`BasePassGBuffer` if:

- Active PS exists.
- `DrawIndexedInstanced` or graphics drawcall.
- Multiple render targets bound, usually >= 3.
- PS reflection outputs `SV_Target0..3`.
- No compute dispatch dimensions.
- No evidence that this is a lighting fullscreen pass.

`NaniteGBufferCompute` if:

- Active CS exists and writes GBuffer UAVs.
- No active PS material pass.

### `renderdoc_material_runtime_evidence.json`

Purpose: one compact top-level file for Unity reconstruction Agent.

Required fields:

```json
{
  "Schema": "ue-renderdoc-material-runtime-evidence/v1",
  "ReadFirst": [
    "renderdoc_runtime_overlay.md",
    "renderdoc_material_runtime_evidence.md",
    "renderdoc_shader_io_summary.json",
    "renderdoc_cpd_summary.json",
    "renderdoc_svt_summary.json",
    "renderdoc_gbuffer_outputs.json"
  ],
  "Facts": [],
  "Assumptions": [],
  "DoNotPort": [
    "SVT page table indirection",
    "SVT feedback UAV",
    "UE renderer-only GBuffer packing not required by Unity material shader"
  ],
  "UseForUnityReconstruction": []
}
```

## Implementation Phases

### Phase 1 - Refactor Current Script Into Small Passes

Current script is standalone and should remain standalone.

Add internal pass functions:

```text
collect_shader_io_summary(...)
collect_svt_summary(...)
collect_cpd_summary(...)
collect_gbuffer_outputs(...)
collect_material_runtime_evidence(...)
write_extra_markdown(...)
```

Do not move to C# yet. Python is faster to iterate and already fits the current goal workflow.

### Phase 2 - Shader IO Summary

Inputs:

```text
Shaders/*/reflection.json
Shaders/*/shader.json
drawcall.json
```

Tasks:

- Normalize stage names.
- Extract input/output signatures.
- Preserve semantic name, register index, channel mask, component count, type.
- Mark important semantics:
  - `COLOR`
  - `TEXCOORD*`
  - `PRIMITIVE_ID`
  - `CUSTOM_DATA_OFFSET`
  - `CUSTOM_DATA_COUNT`
  - `SV_Target*`
  - `SV_Position`
- Output JSON + Markdown.

### Phase 3 - SVT Classifier

Inputs:

```text
Textures/Metadata/textures.json
ResourceBuffers/resource_buffers.json
Shaders/ps/reflection.json
Shaders/ps/disassembly_native_*.txt
```

Tasks:

- Identify integer page table textures.
- Identify compressed physical texture candidates.
- Identify repeated resource ids across multiple registers.
- Identify UAV/feedback resources.
- Add clear Unity handling note:

```text
SVT runtime is renderer-only. Unity reconstruction should expose direct texture properties and not port UE page-table lookup.
```

### Phase 4 - CPD Dataflow And Value Summary

Inputs:

```text
Shaders/vs/reflection.json
Shaders/vs/disassembly_native_*.txt
Shaders/ps/reflection.json
Shaders/ps/disassembly_native_*.txt
ResourceBuffers/resource_buffers.json
ResourceBuffers/*.raw.bin
Mesh/mesh_postvs.json
Mesh/postvs_vertex_buffer.bin
```

Tasks:

- Detect CPD semantics from reflection.
- Parse VS snippets that write `CUSTOM_DATA_OFFSET` and `CUSTOM_DATA_COUNT`.
- Parse PS snippets that read `CUSTOM_DATA_OFFSET`, `CUSTOM_DATA_COUNT`, and `StructuredBuffer0`.
- Extract compact access patterns:
  - `count > N`
  - `Load(offset)`
  - `Load(offset + K)`
  - component x/y/z/w use
- Try to recover actual offset:
  - Prefer post-VS decoded semantic data if available.
  - Otherwise inspect post-VS raw buffer only if layout is known and small enough.
  - Otherwise output dataflow-only partial summary.
- Decode CPD raw values only when offset and stride are known.

Output must include both:

- `CpdPathEvidence`: proven dataflow.
- `CpdValueEvidence`: actual decoded values, or explicit reason why not available.

### Phase 5 - GBuffer/MRT Summary

Inputs:

```text
Textures/Metadata/textures.json
pipeline_state.json
Shaders/ps/reflection.json
drawcall.json
```

Tasks:

- Map PS `SV_TargetN` to runtime `rtN`.
- Record format, dimensions, resource id, file path.
- Detect depth target.
- Classify drawcall role using MRT count, shader stage, topology, and output signatures.
- Add Unity Deferred handling guidance.

### Phase 6 - Focused DXIL Snippets

Current `interesting_dxil_snippets.md` should be split or extended with section tags:

```text
## Shader IO
## CPD Path
## SVT Runtime
## Texture Samples
## GBuffer Outputs
## UAV / Feedback
```

Snippet selection rules:

- Include lines around declarations:
  - `StructuredBuffer`
  - `Texture2D`
  - `RWStructuredBuffer`
  - `cbuffer`
  - `SamplerState`
- Include lines around CPD:
  - `CUSTOM_DATA_OFFSET`
  - `CUSTOM_DATA_COUNT`
  - `StructuredBuffer0.Load`
- Include lines around SVT:
  - integer `Texture2D<uint4>.Load`
  - page table textures
  - feedback UAV write
- Include lines around outputs:
  - `_OUT.SV_TargetN`

Hard cap:

- Keep total snippets under the existing `--max-snippet-lines`.
- If over budget, prioritize:
  1. CPD path
  2. GBuffer outputs
  3. SVT resource declarations
  4. texture samples
  5. cbuffer loads

### Phase 7 - Bundle Integration

When `--bundle-dir` is provided:

- Write outputs into `<Bundle>/analysis/renderdoc`.
- Update or rely on subsequent:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- --context-only "<Bundle>" --verbose
```

Agent docs should mention new files if present:

```text
analysis/renderdoc/renderdoc_material_runtime_evidence.md
analysis/renderdoc/renderdoc_cpd_summary.json
analysis/renderdoc/renderdoc_svt_summary.json
analysis/renderdoc/renderdoc_gbuffer_outputs.json
analysis/renderdoc/renderdoc_shader_io_summary.json
```

Read order:

1. cooked `analysis/ai_context_pack.md/json`
2. cooked `analysis/reconstruction_entrypoints.json`
3. cooked `analysis/module_formula_evidence.json`
4. RenderDoc `renderdoc_material_runtime_evidence.md/json`
5. RenderDoc `renderdoc_cpd_summary.json`
6. RenderDoc `renderdoc_svt_summary.json`
7. RenderDoc `renderdoc_gbuffer_outputs.json`
8. raw selected DXIL only if needed

### Phase 8 - Verification Fixtures

Use at least two fixtures:

1. PS GBuffer drawcall:

```text
K:\WorkSpace\ShaderRerverseWP\MI_CG_RockSmooth_01a\RenderDocCapture\EID_6250_ID3D12GraphicsCommandList_DrawIndexedInstanced
```

Expected:

```text
DrawcallRole = BasePassGBuffer
SVT Detected = true
CPD Detected = true
PS outputs include SV_Target0..3
Render targets include rt0..rt4 + depth
Texture inputs include t2..t17
```

2. Existing Nanite/CS or runtime-only fixture.

Expected:

```text
DrawcallRole = NaniteGBufferCompute or RendererRuntime
PS-specific CPD summary = not_found or skipped
Existing output behavior remains partial/success, not failed
```

Verification commands:

```powershell
python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\renderdoc_compact_summary.py `
  --renderdoc-dir "K:\WorkSpace\ShaderRerverseWP\MI_CG_RockSmooth_01a\RenderDocCapture\EID_6250_ID3D12GraphicsCommandList_DrawIndexedInstanced" `
  --bundle-dir "K:\WorkSpace\ShaderRerverseWP\MI_CG_RockSmooth_01a\MI_CG_RockSmooth_01a.bundle"
```

Then:

```powershell
python -c "import json, pathlib; p=pathlib.Path(r'K:\WorkSpace\ShaderRerverseWP\MI_CG_RockSmooth_01a\MI_CG_RockSmooth_01a.bundle\analysis\renderdoc'); [json.load(open(f, encoding='utf-8')) for f in p.glob('*.json')]; print('RenderDoc compact JSON: OK')"
```

If bundle is available:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --context-only "K:\WorkSpace\ShaderRerverseWP\MI_CG_RockSmooth_01a\MI_CG_RockSmooth_01a.bundle" `
  --verbose
```

## Acceptance Criteria

1. Existing compact summary workflow still works with old inputs.
2. PS GBuffer drawcall produces:
   - `renderdoc_shader_io_summary.json/md`
   - `renderdoc_svt_summary.json/md`
   - `renderdoc_cpd_summary.json/md`
   - `renderdoc_gbuffer_outputs.json/md`
   - `renderdoc_material_runtime_evidence.json/md`
3. `renderdoc_svt_summary.json` correctly marks SVT runtime as not directly ported to Unity.
4. `renderdoc_cpd_summary.json` proves VS-to-PS CPD path when reflection/disassembly contains it.
5. If CPD raw values cannot be decoded safely, output is `partial` with a specific reason, not a false empty value.
6. `renderdoc_gbuffer_outputs.json` maps `SV_TargetN` to `rtN` and records runtime formats.
7. `interesting_dxil_snippets.md` includes focused CPD/SVT/GBuffer snippets without full DXIL dumps.
8. Generated Agent docs/read order point future Agents to compact RenderDoc files before raw RenderDoc data.

## Risks And Limits

- RenderDoc reflection can still omit UE source parameter names.
- Texture resource ids and runtime names do not prove UE material parameter identity.
- SVT physical texture candidates may map many `t#` registers to the same atlas resource.
- CPD actual values may require post-VS semantic decoding; raw buffer alone is not enough if offset is unknown.
- Shipping builds may optimize shader structure, so dataflow patterns must stay evidence-based.

## Recommended Next Task Document

After this plan is accepted, create:

```text
Doc/UE_RenderDoc_Compact_Summary_PS_SVT_CPD_Tasks.md
```

The task document should split this plan into implementation tasks and verification checkpoints.
