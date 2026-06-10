# UE RenderDoc Compact Summary PS/SVT/CPD Tasks

Status: task breakdown for `UE_RenderDoc_Compact_Summary_PS_SVT_CPD_Plan.md`.

## Scope

Implement the plan on the FModel side only.

RenderDoc/qrenderdoc remains a raw drawcall exporter. It should not run CPD/SVT/GBuffer analysis and should not generate compact CPD reports.

Primary implementation target:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Tools/renderdoc_compact_summary.py
```

Optional internal helper modules may be added under:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Tools/shader_reconstruction/
```

but the Agent-facing stable outputs must be emitted by `renderdoc_compact_summary.py`.

## Task List

### 1. Preserve Existing Workflow

Status: done

- Keep current CLI arguments working:
  - `--root-dir`
  - `--renderdoc-dir`
  - `--bundle-dir`
  - `--out-dir`
  - `--max-snippet-lines`
  - `--max-texture-bindings`
- Keep existing output files:
  - `renderdoc_runtime_overlay.json/md`
  - `renderdoc_shader_match.json`
  - `renderdoc_texture_slot_map.json`
  - `renderdoc_sampler_slot_map.json`
  - `renderdoc_cbuffer_value_usage.json`
  - `renderdoc_runtime_outputs.json`
  - `renderdoc_drawcall_role.json`
  - `interesting_dxil_snippets.md`
  - `agent_notes.md`
- Ensure older D3D11/D3D12 and Nanite/CS captures still produce success/partial output instead of failing.

### 2. Refactor Script Into Pass Functions

Status: done

- Add clear internal pass functions:
  - `collect_shader_io_summary(...)`
  - `collect_svt_summary(...)`
  - `collect_cpd_summary(...)`
  - `collect_gbuffer_outputs(...)`
  - `collect_material_runtime_evidence(...)`
  - `build_extra_markdown(...)`
- Keep the script standalone.
- Avoid large new dependencies.
- Keep all JSON writing through existing `write_json(...)` style helpers.

### 3. Shader IO Summary

Status: done

Output:

```text
renderdoc_shader_io_summary.json
renderdoc_shader_io_summary.md
```

Inputs:

```text
Shaders/*/reflection.json
Shaders/*/shader.json
drawcall.json
```

Tasks:

- Normalize stage names: `vs`, `ps`, `cs`, etc.
- Extract stage entry point, encoding, and resource id.
- Extract input signature:
  - semantic name
  - semantic index
  - register index
  - channel masks
  - component count
  - type
  - system value
- Extract output signature.
- Highlight material-relevant semantics:
  - `COLOR`
  - `TEXCOORD*`
  - `PRIMITIVE_ID`
  - `CUSTOM_DATA_OFFSET`
  - `CUSTOM_DATA_COUNT`
  - `SV_Target*`
  - `SV_Position`
- Add `EvidenceLevel=runtime_reflection`.

Acceptance:

- PS GBuffer fixture reports `CUSTOM_DATA_OFFSET`, `CUSTOM_DATA_COUNT`, `PRIMITIVE_ID`, `COLOR`, and `SV_Target0..3`.

### 4. SVT Runtime Summary

Status: done

Output:

```text
renderdoc_svt_summary.json
renderdoc_svt_summary.md
```

Inputs:

```text
Textures/Metadata/textures.json
ResourceBuffers/resource_buffers.json
Shaders/ps/reflection.json
Shaders/ps/disassembly_native_*.txt
```

Tasks:

- Detect SVT presence from runtime resource patterns.
- Classify page table texture candidates:
  - integer formats such as `R32G32_UINT`, `R32_UINT`
  - repeated small/medium render target resources
  - shader-read/color-target resource flags
- Classify physical texture candidates:
  - compressed formats such as `BC1`, `BC3`, `BC5`, `BC7`
  - large dimensions
  - repeated resource id across multiple registers
- Classify feedback/runtime UAV candidates:
  - `RWStructuredBuffer`
  - `RWTexture`
  - `u#`
  - disassembly lines containing `Interlocked`, `Store`, feedback-like writes
- Output Unity handling:

```text
Do not port SVT indirection. Use direct Unity texture properties and mark UE VT runtime as abstracted.
```

Acceptance:

- PS GBuffer fixture marks `Detected=true`.
- Page table candidates include integer `t#` resources.
- Physical candidates include large BC compressed texture resources.
- Feedback/UAV candidates include PS `u0` when present.

### 5. CPD Summary Analyzer

Status: done

Output:

```text
renderdoc_cpd_summary.json
renderdoc_cpd_summary.md
```

Inputs:

```text
Shaders/vs/reflection.json
Shaders/vs/disassembly_native_*.txt
Shaders/ps/reflection.json
Shaders/ps/disassembly_native_*.txt
ResourceBuffers/resource_buffers.json
ResourceBuffers/*.raw.bin
Mesh/vertex_input_layout.json
Mesh/mesh_postvs.json
Mesh/postvs_vertex_buffer.bin
```

Tasks:

- Detect CPD semantics from VS and PS reflection:
  - `PRIMITIVE_ID`
  - `CUSTOM_DATA_OFFSET`
  - `CUSTOM_DATA_COUNT`
- Detect VS-to-PS path:
  - VS output includes CPD semantics.
  - PS input consumes the same semantics.
- Parse focused DXIL/native disassembly lines:
  - `_IN.CUSTOM_DATA_OFFSET`
  - `_IN.CUSTOM_DATA_COUNT`
  - `StructuredBuffer0.Load(offset)`
  - `StructuredBuffer0.Load(offset + K)`
  - count guards such as `CUSTOM_DATA_COUNT > N`
- Resolve runtime buffers:
  - map `StructuredBuffer0` to `ResourceBuffers/resource_buffers.json`
  - capture stage, register, resource id, stride/element size, raw path
- Extract access patterns:
  - base offset expression
  - offset additions
  - component x/y/z/w usage
  - count conditions
- Decode raw float4 windows only when offset and stride are proven.
- If actual per-instance offset cannot be recovered, output:

```json
{
  "Status": "partial",
  "Reason": "CPD path is proven, but actual per-instance CUSTOM_DATA_OFFSET value was not extracted."
}
```

Acceptance:

- PS GBuffer fixture reports `Detected=true`.
- VS-to-PS path is proven.
- PS `StructuredBuffer0.Load(_206)` style access is summarized.
- No fake empty CPD values are emitted when offset is unknown.

### 6. Optional UE GPUScene Layout Hints

Status: done

Output:

```text
renderdoc_cpd_summary.json.Layout
```

Inputs:

```text
Tools/shader_reconstruction/layouts/UE5_5_GPUScene.example.json
Tools/shader_reconstruction/layouts/UE5_6_GPUScene.example.json
```

Tasks:

- Add optional layout hint files on the FModel side only.
- Use layout hints to improve CPD/resource candidate scoring.
- Do not require these hints for basic CPD path detection.
- Mark layout source as:
  - `UE5_5_GPUScene`
  - `UE5_6_GPUScene`
  - `custom`
  - `unknown`

Acceptance:

- Missing layout hints do not fail compact summary.
- If hints are present and used, the chosen layout is recorded in `renderdoc_cpd_summary.json`.

### 7. GBuffer/MRT Output Summary

Status: done

Output:

```text
renderdoc_gbuffer_outputs.json
renderdoc_gbuffer_outputs.md
```

Inputs:

```text
Textures/Metadata/textures.json
pipeline_state.json
Shaders/ps/reflection.json
drawcall.json
```

Tasks:

- Map PS `SV_TargetN` outputs to runtime `rtN` outputs.
- Record:
  - slot
  - semantic
  - format
  - width/height
  - resource id
  - exported file path
- Record depth target when present.
- Improve drawcall role classifier:
  - `BasePassGBuffer`
  - `NaniteGBufferCompute`
  - `RendererRuntime`
  - `Unknown`
- Classify `BasePassGBuffer` when:
  - active PS exists
  - drawcall is graphics draw
  - multiple MRT outputs exist
  - PS outputs `SV_Target0..3`
  - no compute dispatch dimensions
- Add Unity Deferred handling guidance:

```text
Map material surface outputs to Unity URP Deferred-compatible GBuffer code; reuse Unity URP deferred lighting.
```

Acceptance:

- PS GBuffer fixture maps `SV_Target0..3` to runtime `rt0..rt3`.
- Runtime target formats are recorded.
- Role is `BasePassGBuffer` with evidence.

### 8. Material Runtime Evidence Summary

Status: done

Output:

```text
renderdoc_material_runtime_evidence.json
renderdoc_material_runtime_evidence.md
```

Tasks:

- Create one compact top-level summary for Unity reconstruction Agents.
- Include:
  - drawcall role
  - shader stages
  - shader match summary
  - CPD status
  - SVT status
  - GBuffer/MRT status
  - texture slot highlights
  - runtime-only features to abstract
- Include `ReadFirst` list:

```text
renderdoc_runtime_overlay.md
renderdoc_material_runtime_evidence.md
renderdoc_shader_io_summary.json
renderdoc_cpd_summary.json
renderdoc_svt_summary.json
renderdoc_gbuffer_outputs.json
```

- Include `DoNotPort`:
  - SVT page table indirection
  - SVT feedback UAV
  - UE renderer-only GBuffer packing not needed in Unity material shader

Acceptance:

- A fresh Unity Shader reconstruction Agent can read this file before raw RenderDoc exports.

### 9. Focused DXIL Snippets

Status: done

Output:

```text
interesting_dxil_snippets.md
```

Tasks:

- Preserve existing snippet output.
- Add section tags:
  - `Shader IO`
  - `CPD Path`
  - `SVT Runtime`
  - `Texture Samples`
  - `GBuffer Outputs`
  - `UAV / Feedback`
- Prioritize snippets:
  1. CPD path
  2. GBuffer outputs
  3. SVT resource declarations
  4. texture samples
  5. cbuffer loads
- Keep total snippets under `--max-snippet-lines`.
- Do not dump full shader disassembly.

Acceptance:

- PS GBuffer fixture includes lines around:
  - `CUSTOM_DATA_OFFSET`
  - `CUSTOM_DATA_COUNT`
  - `StructuredBuffer0.Load`
  - `SV_Target0..3`
  - SVT page table / physical texture declarations

### 10. Overlay Integration

Status: done

Tasks:

- Add new files to `renderdoc_runtime_overlay.json.RecommendedReadOrder`.
- Add high-level status flags to `renderdoc_runtime_overlay.json`:
  - `ShaderIoSummary`
  - `SvtSummary`
  - `CpdSummary`
  - `GBufferOutputs`
  - `MaterialRuntimeEvidence`
- Update `renderdoc_runtime_overlay.md` to mention the new compact files.
- Keep legacy files valid and parseable.

Acceptance:

- Existing users who only read `renderdoc_runtime_overlay.md/json` can discover the new summaries.

### 11. Agent Workspace Integration

Status: done

Files to update:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/AgentWorkspaceWriter.cs
Doc/UE_RenderDoc_Compact_Summary_Goal.md
```

Tasks:

- Add optional read-first entries when files exist:
  - `analysis/renderdoc/renderdoc_material_runtime_evidence.md`
  - `analysis/renderdoc/renderdoc_material_runtime_evidence.json`
  - `analysis/renderdoc/renderdoc_cpd_summary.json`
  - `analysis/renderdoc/renderdoc_svt_summary.json`
  - `analysis/renderdoc/renderdoc_gbuffer_outputs.json`
  - `analysis/renderdoc/renderdoc_shader_io_summary.json`
- Update Agent instructions:
  - Read compact RenderDoc evidence before raw RenderDoc files.
  - Treat SVT as runtime-only/abstracted.
  - Treat CPD values as runtime facts, not UE source parameter names.
  - Keep Unity target as Unity 6 URP Deferred + DOTS instancing.

Acceptance:

- `--context-only <bundle>` refreshes Agent docs after compact summary files exist.

### 12. Verification Fixture: PS GBuffer Drawcall

Status: done

Fixture:

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

Commands:

```powershell
python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\renderdoc_compact_summary.py `
  --renderdoc-dir "K:\WorkSpace\ShaderRerverseWP\MI_CG_RockSmooth_01a\RenderDocCapture\EID_6250_ID3D12GraphicsCommandList_DrawIndexedInstanced" `
  --out-dir "D:\Tmp\RenderDocCompactTest\MI_CG_RockSmooth_01a"
```

JSON parse:

```powershell
python -c "import json, pathlib; p=pathlib.Path(r'D:\Tmp\RenderDocCompactTest\MI_CG_RockSmooth_01a'); [json.load(open(f, encoding='utf-8')) for f in p.glob('*.json')]; print('RenderDoc compact JSON: OK')"
```

Acceptance:

- All new JSON files exist and parse.
- Markdown summaries exist and are non-empty.
- No full raw disassembly or raw buffer is inlined into JSON/Markdown.

### 13. Verification Fixture: Existing Nanite/CS Or Runtime-Only Capture

Status: done

Tasks:

- Run existing compact summary flow on one older Nanite/CS or runtime-only fixture.
- Confirm:
  - CPD summary is `not_found` or `skipped`.
  - SVT summary may be `not_found` or partial.
  - Existing output remains success/partial.
  - No regression to current workflow.

Acceptance:

- New PS-specific logic does not break old captures.

### 14. Documentation Update

Status: done

Files:

```text
Doc/UE_RenderDoc_Compact_Summary_Goal.md
Doc/UE_RenderDoc_Compact_Summary_PS_SVT_CPD_Plan.md
```

Tasks:

- Update output file list.
- Document RenderDoc/qrenderdoc responsibility boundary:

```text
RenderDoc/qrenderdoc exports raw drawcall data only.
FModel compact summary performs CPD/SVT/GBuffer analysis.
```

- Add a note that SVT should generally not be disabled for primary evidence.
- Add a note that SVT runtime is not ported to Unity material shader.

Acceptance:

- New AI sessions invoking `/Goal D:\Github\FModel\Doc\UE_RenderDoc_Compact_Summary_Goal.md` understand the new outputs.

## Final Acceptance Criteria

- `renderdoc_compact_summary.py` produces PS/SVT/CPD/GBuffer summaries from raw RenderDoc drawcall exports.
- RenderDoc/qrenderdoc remains raw-export-only.
- CPD summary proves path when available and never fabricates values when offset is unknown.
- SVT summary separates page-table/physical/feedback resources and marks them as Unity-side abstraction.
- GBuffer summary maps PS outputs to runtime MRT formats.
- Agent docs direct future reconstruction work to compact summaries before raw RenderDoc files.

## Verification Evidence

Completed on 2026-06-10.

Commands run:

```powershell
python -m py_compile CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\renderdoc_compact_summary.py

python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\renderdoc_compact_summary.py `
  --renderdoc-dir "K:\WorkSpace\ShaderRerverseWP\MI_CG_RockSmooth_01a\RenderDocCapture\EID_6250_ID3D12GraphicsCommandList_DrawIndexedInstanced" `
  --out-dir "D:\Tmp\RenderDocCompactTest\MI_CG_RockSmooth_01a"

python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\renderdoc_compact_summary.py `
  --renderdoc-dir "K:\WorkSpace\ShaderReverse\M_LayerStandard\RenderDocCapture\EID_10965_[0]_arg0_IndirectDispatch_0,_1,_1" `
  --out-dir "D:\Tmp\RenderDocCompactTest\M_LayerStandard_NaniteCS"

python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\renderdoc_compact_summary.py `
  --renderdoc-dir "K:\WorkSpace\ShaderRerverseWP\MI_CG_RockSmooth_01a\RenderDocCapture\EID_6250_ID3D12GraphicsCommandList_DrawIndexedInstanced" `
  --bundle-dir "D:\Tmp\ShaderBundles\MI_CG_RockSmooth_01a.bundle"

dotnet build CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release

dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --context-only "D:\Tmp\ShaderBundles\MI_CG_RockSmooth_01a.bundle" `
  --verbose
```

Observed results:

- PS GBuffer fixture: `DrawcallRole=BasePassGBuffer`, `CPD Status=partial Detected=true`, `SVT Detected=true`, `GBuffer MRT targets=5`, `Depth=D32S8_TYPELESS`.
- Nanite/CS fixture: `DrawcallRole=RendererRuntime / NaniteGBufferCompute`, `CPD Status=not_found Detected=false`, JSON output remained parseable.
- Bundle integration: compact summaries were written to `analysis/renderdoc`, `agent_context.json` read-first included shader IO/CPD/SVT/GBuffer/material runtime evidence, and `--context-only` returned `Verify: OK`.
- C# build succeeded. The environment still reports the pre-existing `cmake` missing warning for optional native build, but `CUE4Parse.ShaderBundleExporter` built successfully.
