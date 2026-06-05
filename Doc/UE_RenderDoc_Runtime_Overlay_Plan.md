# UE + RenderDoc Runtime Overlay Implementation Plan

本文档定义 `CUE4Parse.ShaderBundleExporter` 的下一阶段功能：把 RenderDoc 单个 drawcall 导出的运行时绑定数据，压缩成 AI Agent 友好的 runtime overlay，并与现有 UE cooked shader bundle 的静态分析结果结合。

目标不是把 RenderDoc 全量目录塞进上下文，而是把它变成小而准确的证据层，用来补足 UE cooked 静态流程里的匿名 `t#`、`cb#`、sampler、render target 和 pass role 判断。

## Problem

当前 UE cooked 静态流程已经能输出：

```text
analysis/semantic_binding_map.json
analysis/texture_register_candidates.json
analysis/texture_register_statistics.json
analysis/texture_channel_semantics.json
analysis/uniform_buffer_usage.json
analysis/gbuffer_semantics.json
analysis/ai_context_pack.json
```

但仍有几个弱点：

```text
DXIL texture registers are anonymous: t0/t1/t2...
cbuffer fields are often anonymous: cb0[29], cb5[3]...
UniformExpressionSet order is not a source-level t# binding.
Shader archive metadata often has hashes/indexes, not readable material parameter names.
AI Agent can over-read raw DXIL / RenderDoc exports and mix material logic with renderer runtime logic.
```

RenderDoc drawcall export can provide runtime evidence:

```text
t# -> actual bound resource name/path/format/size
s# -> actual sampler state
cb# -> actual buffer resource and values
VS/PS bytecode and reflection for the captured draw
render targets / depth target
pipeline topology / vertex input layout / post-VS geometry
```

This evidence should improve confidence, but only if compressed into a small overlay.

## Goals

1. Add optional RenderDoc drawcall directory input to `CUE4Parse.ShaderBundleExporter`.
2. Generate compact runtime overlay files inside the existing UE shader bundle.
3. Match RenderDoc shader bytecode/disassembly to cooked bundle shaders when possible.
4. Map runtime texture/sampler bindings to anonymous DXIL registers.
5. Extract only cbuffer values actually used by the shader, not full cbuffer dumps.
6. Classify drawcall role: BasePass/GBuffer, LightingPass, DepthOnly, Shadow, ForwardLit, RendererRuntime, Unknown.
7. Update Agent docs and AI context so a new Agent reads overlay first and raw RenderDoc files only on demand.

## Non-Goals

```text
Do not claim RenderDoc restores UE source material graph node names.
Do not inline full RenderDoc disassembly, full cbuffer CSV, raw buffers, or large textures into AI context.
Do not require RenderDoc export for the existing pure static workflow.
Do not treat lighting/runtime drawcalls as material BasePass evidence unless role classifier supports it.
Do not make M_Character_Teeth-specific or sample-directory-specific assumptions.
```

## CLI

Add optional argument:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --semantic-only D:\ShaderWP\NewTest `
  --renderdoc-drawcall-dir "K:\WorkSpace\Profiler\Test005\EID_3483_ID3D11DeviceContext_DrawIndexedInstanced" `
  --verbose
```

Full export should also support the same option:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --game Subnautica2 `
  --paks "..." `
  --mapping "..." `
  --material "/Game/Materials/_Master/Master/M_Character_Teeth" `
  --out "D:\ShaderWP\M_Character_Teeth.bundle" `
  --renderdoc-drawcall-dir "K:\...\EID_..."
```

If the argument is absent, existing behavior remains unchanged.

## Output Layout

All output goes into the UE bundle:

```text
analysis/renderdoc_runtime_overlay.json
analysis/renderdoc_runtime_overlay.md
analysis/renderdoc_shader_match.json
analysis/renderdoc_texture_slot_map.json
analysis/renderdoc_sampler_slot_map.json
analysis/renderdoc_cbuffer_value_usage.json
analysis/renderdoc_drawcall_role.json
```

`manifest.json`, `analysis/semantic_status.json`, `analysis/ai_context_pack.json`, `AGENTS.md`, and `WORKFLOW.md` should reference these files when generated.

## Core Overlay Schema

`analysis/renderdoc_runtime_overlay.json`:

```json
{
  "Schema": "ue-renderdoc-runtime-overlay/v1",
  "Status": "success|partial|failed",
  "RenderDocDrawcallDirectory": "K:/...",
  "Drawcall": {
    "EventId": 3483,
    "Api": "D3D11",
    "Action": "ID3D11DeviceContext::DrawIndexedInstanced()",
    "Topology": "Triangle List",
    "NumIndices": 4062,
    "NumInstances": 1
  },
  "DrawcallRole": {
    "RoleCandidate": "BasePassGBuffer|LightingPass|DepthOnly|Shadow|ForwardLit|RendererRuntime|Unknown",
    "Confidence": 0.0,
    "Evidence": []
  },
  "ShaderMatch": {
    "PixelShader": {
      "RenderDocShader": "Shaders/ps/raw_shader.bin",
      "MatchedCookedShader": "shaders/005_unknown_idx_46518_group_02843.dxil",
      "MatchKind": "byte_hash|disassembly_hash|reflection_signature|unmatched",
      "Confidence": 0.0
    },
    "VertexShader": {}
  },
  "TextureBindings": [],
  "SamplerBindings": [],
  "ConstantBufferValuesUsed": [],
  "RenderTargets": [],
  "RecommendedReadOrder": [],
  "DoNotReadByDefault": [],
  "KnownLimitations": []
}
```

## Phase 1 - Input Discovery And Validation

Inputs:

```text
drawcall.json
pipeline_state.json
shader_reconstruction_index.json
Shaders/vs/reflection.json
Shaders/ps/reflection.json
Shaders/vs/raw_shader.bin
Shaders/ps/raw_shader.bin
Shaders/vs/disassembly_native_*.txt
Shaders/ps/disassembly_native_*.txt
Shaders/vs/disassembly_hlsl_*.txt
Shaders/ps/disassembly_hlsl_*.txt
Textures/Metadata/textures.json
Textures/Metadata/samplers.json
ConstantBuffers/constant_buffers.json
Mesh/vertex_input_layout.json
Mesh/mesh_postvs.json
ResourceBuffers/resource_buffers.json
```

Validation:

```text
Missing drawcall.json => overlay failed.
Missing pipeline_state.json => overlay partial.
Missing shader files => shader match skipped.
Missing texture metadata => texture overlay skipped.
Missing cbuffer metadata => cbuffer used-value overlay skipped.
```

Implementation:

```text
Exporter/RenderDocOverlay/RenderDocDrawcallReader.cs
Exporter/RenderDocOverlay/RenderDocOverlayPipeline.cs
Models/RenderDocOverlayModels.cs
```

## Phase 2 - Shader Match

Purpose: determine whether the captured RenderDoc VS/PS correspond to shaders already present in the UE cooked bundle.

Match levels:

```text
byte_hash:
  hash RenderDoc Shaders/<stage>/raw_shader.bin and cooked bundle shaders/*.dxil or *.dxbc
  strongest match

disassembly_hash:
  normalize native disassembly declarations/instructions and compare
  strong match when byte containers differ but code is equivalent

reflection_signature:
  compare stage, resource declarations, cbuffer sizes, output signature, instruction counts
  weak/medium match

unmatched:
  keep RenderDoc as runtime evidence only
```

Output:

```text
analysis/renderdoc_shader_match.json
```

Acceptance:

```text
Matched shader includes MatchKind, Confidence, Evidence, SourceFiles.
Unmatched shader still records RenderDoc shader ID, stage, entry point, encoding, and reason.
```

## Phase 3 - Texture And Sampler Slot Overlay

Purpose: convert RenderDoc runtime bindings into compact `t#` and `s#` maps.

Inputs:

```text
Textures/Metadata/textures.json
Textures/Metadata/samplers.json
Shaders/ps/reflection.json
Shaders/ps/disassembly_native_*.txt
analysis/texture_register_candidates.json
analysis/texture_register_statistics.json
parameters/textures.json
```

Output:

```text
analysis/renderdoc_texture_slot_map.json
analysis/renderdoc_sampler_slot_map.json
```

Example texture entry:

```json
{
  "Stage": "ps",
  "Register": "t000",
  "RuntimeName": "A_Texture 50",
  "ResourceId": "ResourceId::2128",
  "TextureType": "Texture 2D",
  "Format": "BC1_SRGB",
  "Size": "1024x1024x1",
  "File": "Textures/Inputs/tex_in__ps__t000__A_Texture_50__ResourceId_2128.dds",
  "CookedCandidates": [
    {
      "ParameterName": "BC",
      "Confidence": 0.27,
      "EvidenceLevel": "static_candidate_plus_runtime_binding"
    }
  ],
  "Evidence": [
    "RenderDoc captured a concrete resource bound to ps t000.",
    "Static UE bundle candidate is still not a source-level material graph binding."
  ]
}
```

Texture confidence rules:

```text
+0.45 concrete RenderDoc t# resource binding exists
+0.20 runtime resource name/path matches UE texture asset name or exported texture path
+0.15 format/size/sRGB matches UE texture metadata
+0.10 static texture_register_statistics top candidate agrees
-0.30 if drawcall role is LightingPass/RendererRuntime and texture is shadow/depth/DBuffer/light-list
```

Sampler output should include:

```text
filter
address_u/v/w
compare_function
LOD range
Unity sampler declaration candidate
```

## Phase 4 - CBuffer Used-Value Overlay

Purpose: avoid dumping full cbuffer files into AI context. Only emit values referenced by shader instructions.

Inputs:

```text
Shaders/ps/disassembly_native_*.txt
Shaders/vs/disassembly_native_*.txt
ConstantBuffers/constant_buffers.json
ConstantBuffers/<stage>/*.variables.csv
Shaders/<stage>/reflection.json
analysis/uniform_buffer_usage.json
```

Parse patterns:

```text
cb0[29].w
cb5[3].xyz
cb4[r0.y + 1].xyz
cb1[r0.z + 0].x
```

Output:

```text
analysis/renderdoc_cbuffer_value_usage.json
```

Entry:

```json
{
  "Stage": "ps",
  "Register": "b005",
  "CBufferName": "Buffer-512-512",
  "Index": 3,
  "Components": "xyz",
  "StaticAccess": true,
  "DynamicAccess": false,
  "Value": [0.141509414, 0.129780531, 0.11414203],
  "UsedAtLines": [139, 140],
  "CookedParameterCandidates": [],
  "Evidence": [
    "Native disassembly references cb5[3].xyz.",
    "Value was read from RenderDoc decoded cbuffer CSV."
  ]
}
```

Dynamic indexing policy:

```text
Immediate indexed cb#[] access:
  include exact value.

Dynamic indexed cb#[r# + N]:
  include access pattern, range hints, and cbuffer resource identity.
  do not expand large ranges unless explicitly requested.
```

## Phase 5 - Drawcall Role Classifier

Purpose: prevent AI Agent from mistaking renderer runtime/light pass logic for material surface logic.

Signals:

```text
BasePassGBuffer:
  multiple MRTs or GBuffer-like render targets
  material texture inputs
  depth target active
  no shadowmap-heavy sampling

LightingPass:
  output is CameraTarget / HDR target
  samples depth, shadow maps, light volume, DBuffer, GBuffer
  large light/tile/cluster buffers present

DepthOnly:
  depth output, no color target or simple color target
  minimal pixel shader resources

Shadow:
  shadowmap render target/depth target
  shadow caster shader patterns

ForwardLit:
  material textures + lighting/shadow sampling + direct color target
  no GBuffer MRT output

RendererRuntime:
  tile/z-bin/cluster/light-list/feedback/composite-heavy draw
```

Output:

```text
analysis/renderdoc_drawcall_role.json
```

Role affects AI read policy:

```text
BasePassGBuffer:
  use as strong material reconstruction evidence.

LightingPass:
  use for final visual validation and lighting policy.
  do not treat as material graph source.

RendererRuntime:
  document and abstract.
```

## Phase 6 - Compact Markdown Overlay

Generate:

```text
analysis/renderdoc_runtime_overlay.md
```

Content:

```text
Drawcall summary
Role candidate and confidence
Shader match summary
Texture slot table
Sampler slot table
Top cbuffer used values
Render target/depth target summary
Recommended read order
Do-not-read-by-default list
Known limitations
```

This file should be the first RenderDoc-related file an Agent reads.

## Phase 7 - Agent Workspace Integration

Update generated bundle docs:

```text
AGENTS.md
WORKFLOW.md
NEXT_TASK.md
PROMPT_NEXT_SESSION.md
agent_context.json
analysis/ai_context_pack.json
analysis/ai_context_pack.md
```

Recommended read order:

```text
1. AGENTS.md
2. analysis/ai_context_pack.md
3. analysis/semantic_binding_map.json
4. analysis/texture_register_statistics.json
5. analysis/renderdoc_runtime_overlay.md
6. analysis/renderdoc_runtime_overlay.json
7. analysis/renderdoc_texture_slot_map.json
8. analysis/renderdoc_cbuffer_value_usage.json
```

Do not read by default:

```text
RenderDoc Shaders/*/full disassembly
RenderDoc full pipeline_state.json
RenderDoc full cbuffer JSON/CSV
RenderDoc raw buffers
large DDS/PNG textures
```

Open raw files only when the overlay points to a specific unresolved evidence question.

## Phase 8 - Bundle Verification

`--verify-only` should validate:

```text
If renderdoc_runtime_overlay.json exists, referenced RenderDoc source files exist or are explicitly marked external.
Overlay JSON scores are in [0, 1].
Overlay markdown exists.
Agent read order includes overlay markdown before raw RenderDoc files.
No overlay file inlines large disassembly, full cbuffer dumps, raw buffer bytes, or large texture data.
Drawcall role and shader match include Evidence and SourceFiles.
```

## Acceptance Tests

For the sample directory:

```text
K:\WorkSpace\Profiler\Test005\EID_3483_ID3D11DeviceContext_DrawIndexedInstanced
```

Expected:

```text
renderdoc_runtime_overlay.json exists
renderdoc_runtime_overlay.md exists
renderdoc_texture_slot_map.json contains ps t000..t013
renderdoc_sampler_slot_map.json contains ps s000..s011
renderdoc_cbuffer_value_usage.json includes immediate cb access such as cb5[3].xyz
renderdoc_drawcall_role.json classifies the sample as LightingPass or RendererRuntime/ForwardLit-like, not BasePassGBuffer, because it samples shadow/depth/DBuffer/light-volume resources and writes CameraTarget
verify-only returns Verify: OK
```

For a UE material BasePass/GBuffer capture:

```text
RoleCandidate should be BasePassGBuffer with medium/high confidence.
Texture overlay should improve t# candidate confidence without claiming source material graph binding.
Unity Agent should use overlay as runtime binding evidence after static UE semantic files.
```

## Risks

```text
RenderDoc resource names can be engine/runtime names, not authoring asset names.
A captured draw may be the wrong pass for material reconstruction.
Multiple materials can share a shader permutation; shader byte match alone is not material identity.
Dynamic cbuffer indexing can hide large ranges; only immediate indexes should emit concrete values by default.
RenderDoc HLSL decompiler can be wrong; native disassembly remains ground truth.
```

## Recommended Next Task

Create `Doc/UE_RenderDoc_Runtime_Overlay_Tasks.md` and split this plan into implementation tasks:

```text
Task 01 - CLI and options
Task 02 - RenderDoc directory reader
Task 03 - shader match pass
Task 04 - texture/sampler slot maps
Task 05 - cbuffer used-value analyzer
Task 06 - drawcall role classifier
Task 07 - overlay markdown writer
Task 08 - Agent docs / AI context integration
Task 09 - verifier updates
Task 10 - sample NewTest validation
```
