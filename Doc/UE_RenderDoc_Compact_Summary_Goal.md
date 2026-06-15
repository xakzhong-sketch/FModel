# Goal: Generate RenderDoc Compact Runtime Summary

Objective: use `CUE4Parse.ShaderBundleExporter` tools in `D:\Github\FModel` to convert one RenderDoc current-drawcall export directory into compact AI-friendly runtime evidence for later Unity shader reconstruction.

Do not reconstruct a Unity shader in this task. Only read the RenderDoc export, generate compact summary files, optionally match RenderDoc shader bytecode against an existing UE cooked shader bundle, and report the result.

Responsibility boundary:

```text
RenderDoc/qrenderdoc exports raw drawcall data only.
FModel renderdoc_compact_summary.py performs CPD/SVT/GBuffer/shader IO compact analysis.
```

Do not add or run CPD/SVT/GBuffer analysis inside the RenderDoc export tool. If Streaming Virtual Texturing is present, keep it as runtime evidence and do not try to port UE page-table indirection or feedback UAV writes into the Unity material shader.

## How To Invoke This Goal

Preferred forms:

```text
/Goal D:\Github\FModel\Doc\UE_RenderDoc_Compact_Summary_Goal.md

/Goal D:\Github\FModel\Doc\UE_RenderDoc_Compact_Summary_Goal.md 当前目录

/Goal D:\Github\FModel\Doc\UE_RenderDoc_Compact_Summary_Goal.md Root=K:\WorkSpace\ShaderReverse\M_LayerStandard

/Goal D:\Github\FModel\Doc\UE_RenderDoc_Compact_Summary_Goal.md RenderDoc=K:\WorkSpace\ShaderReverse\M_LayerStandard\M_LayerStandard.bundle\RenderDocCapture\EID_10965_[0]_arg0_IndirectDispatch_0,_1,_1 Bundle=K:\WorkSpace\ShaderReverse\M_LayerStandard\M_LayerStandard.bundle

/Goal D:\Github\FModel\Doc\UE_RenderDoc_Compact_Summary_Goal.md 给 K:\...\EID_10965_[0]_arg0_IndirectDispatch_0,_1,_1 生成 compact summary

/Goal D:\Github\FModel\Doc\UE_RenderDoc_Compact_Summary_Goal.md RenderDoc=K:\...\EID_xxx Bundle=K:\...\Material.bundle Out=K:\...\Material.bundle\analysis\renderdoc
```

Argument rules:

```text
No explicit arguments:
  Treat the current shell directory as Root.
  Auto-detect zero or one *.bundle directory under Root.
  If a bundle is resolved, prefer exactly one RenderDoc current-drawcall export under <Bundle>\RenderDocCapture.
  Legacy Root\RenderDocCapture detection is allowed only when exactly one bundle exists or Bundle=... is explicit.

Root=...
  Convenience mode for a material work directory.
  Auto-detect Bundle first, then prefer Bundle\RenderDocCapture.

RenderDoc=...
  Required unless the user clearly provides a RenderDoc current-drawcall export directory in natural language.
  The directory must contain drawcall.json.

Bundle=...
  Optional existing UE cooked shader bundle directory.
  If present, use it for SHA256 byte-hash matching against bundle/shaders/*.dxil, *.dxbc, and *.bin.
  If absent, skip cooked shader matching and generate runtime-only compact evidence.

Out=...
  Optional output directory.
  If provided, write compact summary files there.
  If omitted and Bundle=... is provided, write to <Bundle>\analysis\renderdoc.
  If omitted and Bundle=... is absent, write to <RenderDoc>\CompactSummary.
```

If the provided RenderDoc directory does not contain `drawcall.json`, do not guess. Report the missing file and ask for the correct current-drawcall export directory.

If auto-detection finds multiple RenderDoc drawcall exports under `Bundle\RenderDocCapture`, list the candidates and ask the user to provide `RenderDoc=...`.

If auto-detection finds multiple `*.bundle` directories, list the candidates and ask the user to provide `Bundle=...`.

In a multi-bundle workspace, do not attach workspace-root RenderDoc data to a bundle by guesswork. New RenderDoc drawcall exports should be placed under the target bundle:

```text
<Bundle>\RenderDocCapture\EID_...
```

If `Bundle=...` is provided but the directory does not contain `shaders`, continue without shader matching only if the user explicitly wants runtime-only output. Otherwise report the mismatch.

## Project Directory

If this goal was invoked from a material workspace directory, for example:

```text
K:\WorkSpace\ShaderReverse\M_LayerStandard
```

record that current directory as `ROOT_DIR` before changing directories.

Then run the tool from the FModel repo, or call the script by absolute path.

FModel repo:

```powershell
cd D:\Github\FModel
```

Script:

```text
CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\renderdoc_compact_summary.py
D:\Github\FModel\CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\renderdoc_compact_summary.py
```

## Expected RenderDoc Export Inputs

High-priority files:

```text
drawcall.json
pipeline_state.json
manifest.json
shader_reconstruction_index.json
AGENTS.md
UnityShaderReconstructionGoal.md
UnityShaderReconstructionWorkflow.md
Textures/Metadata/textures.json
Textures/Metadata/samplers.json
Shaders/*/reflection.json
Shaders/*/raw_shader.bin
Shaders/*/disassembly_native_*.txt
ConstantBuffers/constant_buffers.json
ResourceBuffers/resource_buffers.json
```

The script can still produce partial output if optional files are missing, but `drawcall.json` is required.

## Command

Replace `RENDERDOC_DIR`, `BUNDLE_DIR`, and `OUT_DIR`:

Simple current-directory mode:

```powershell
python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\renderdoc_compact_summary.py `
  --root-dir "CURRENT_MATERIAL_WORK_DIR"
```

Example from a material workspace:

```powershell
python D:\Github\FModel\CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\renderdoc_compact_summary.py `
  --root-dir "K:\WorkSpace\ShaderReverse\M_LayerStandard"
```

If the current shell directory is already the material workspace:

```powershell
python D:\Github\FModel\CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\renderdoc_compact_summary.py `
  --root-dir "."
```

Explicit mode:

```powershell
python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\renderdoc_compact_summary.py `
  --renderdoc-dir "RENDERDOC_DIR" `
  --bundle-dir "BUNDLE_DIR" `
  --out-dir "OUT_DIR"
```

If no UE cooked bundle is available:

```powershell
python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\renderdoc_compact_summary.py `
  --renderdoc-dir "RENDERDOC_DIR" `
  --out-dir "OUT_DIR"
```

Example:

```powershell
python CUE4Parse\CUE4Parse.ShaderBundleExporter\Tools\renderdoc_compact_summary.py `
  --renderdoc-dir "K:\WorkSpace\ShaderReverse\M_LayerStandard\M_LayerStandard.bundle\RenderDocCapture\EID_10965_[0]_arg0_IndirectDispatch_0,_1,_1" `
  --bundle-dir "K:\WorkSpace\ShaderReverse\M_LayerStandard\M_LayerStandard.bundle" `
  --out-dir "K:\WorkSpace\ShaderReverse\M_LayerStandard\M_LayerStandard.bundle\analysis\renderdoc"
```

Expected console result:

```text
RenderDoc compact summary: ...
Status: success|partial
Shader stages: ...
Shader match: ...
```

If `Bundle=...` was provided or auto-detected, refresh the bundle Agent docs after the compact summary is generated:

```powershell
cd D:\Github\FModel

dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --context-only "BUNDLE_DIR" `
  --verbose
```

Expected result:

```text
Context-only: completed
Agent workspace docs: updated
Verify: OK
```

Default output rules:

```text
If a bundle is auto-detected or provided:
  <Bundle>\analysis\renderdoc

If no bundle exists:
  <RenderDoc>\CompactSummary
```

## Output Files

The output directory should contain:

```text
renderdoc_runtime_overlay.md
renderdoc_runtime_overlay.json
renderdoc_material_runtime_evidence.md
renderdoc_material_runtime_evidence.json
renderdoc_shader_io_summary.md
renderdoc_shader_io_summary.json
renderdoc_cpd_summary.md
renderdoc_cpd_summary.json
renderdoc_svt_summary.md
renderdoc_svt_summary.json
renderdoc_gbuffer_outputs.md
renderdoc_gbuffer_outputs.json
renderdoc_shader_match.json
renderdoc_texture_slot_map.json
renderdoc_sampler_slot_map.json
renderdoc_cbuffer_value_usage.json
renderdoc_runtime_outputs.json
renderdoc_drawcall_role.json
interesting_dxil_snippets.md
agent_notes.md
```

## Verify Output

Run a JSON parse check:

```powershell
python -c "import json, pathlib; p=pathlib.Path(r'OUT_DIR'); [json.load(open(f, encoding='utf-8')) for f in p.glob('*.json')]; print('RenderDoc compact summary JSON: OK')"
```

Also inspect the first lines of the markdown summary:

```powershell
Get-Content -LiteralPath "OUT_DIR\renderdoc_runtime_overlay.md" -TotalCount 80
```

Expected findings:

```text
Schema: ue-renderdoc-runtime-overlay/v1
Status: success or partial
Drawcall Role is present
Shader Match table is present
Runtime Outputs table is present
Texture Bindings table is present
```

## Agent Read Order After Export

For later Unity reconstruction, a new Agent should read:

```text
RenderDoc export AGENTS.md
RenderDoc export shader_reconstruction_index.json
OUT_DIR/renderdoc_runtime_overlay.md
OUT_DIR/renderdoc_runtime_overlay.json
OUT_DIR/renderdoc_material_runtime_evidence.md
OUT_DIR/renderdoc_material_runtime_evidence.json
OUT_DIR/renderdoc_shader_io_summary.json
OUT_DIR/renderdoc_cpd_summary.json
OUT_DIR/renderdoc_svt_summary.json
OUT_DIR/renderdoc_gbuffer_outputs.json
OUT_DIR/renderdoc_shader_match.json
OUT_DIR/renderdoc_texture_slot_map.json
OUT_DIR/renderdoc_runtime_outputs.json
OUT_DIR/interesting_dxil_snippets.md
UE bundle analysis/ai_context_pack.md, if Bundle=... was provided
UE bundle analysis/reconstruction_entrypoints.json, if Bundle=... was provided
```

If the compact summary was written into a UE bundle and `--context-only` was run, the normal bundle reconstruction flow remains:

```text
cd BUNDLE_DIR
/Goal PROMPT_NEXT_SESSION.md
```

The generated `PROMPT_NEXT_SESSION.md`, `AGENTS.md`, `WORKFLOW.md`, and `agent_context.json` support both cases:

```text
With analysis/renderdoc/renderdoc_runtime_overlay.md:
  use RenderDoc compact files as optional runtime evidence after cooked compact context.

Without analysis/renderdoc/renderdoc_runtime_overlay.md:
  use the pure cooked static workflow.
```

Do not start by reading:

```text
RenderDoc full disassembly files
RenderDoc raw shader bytecode
RenderDoc full cbuffer CSV files
RenderDoc raw buffer dumps
large DDS/PNG textures
all cooked shader variants
```

Raw RenderDoc files should only be opened for specific evidence questions after the compact overlay is read.

## Interpretation Rules

Use these rules in the final report:

```text
If ShaderMatch.MatchKind is byte_hash:
  RenderDoc shader bytecode exactly matches a cooked bundle shader.
  Runtime evidence can be attached strongly to that cooked shader.

If ShaderMatch.MatchKind is unmatched:
  RenderDoc shader should be treated as runtime evidence only.
  Do not override cooked bundle reconstruction_entrypoints.json.

If DrawcallRole.SpecificRole is NaniteGBufferCompute:
  Treat it as UE renderer/Nanite GBuffer compute evidence.
  Use it to validate output formats, UAV writes, and deferred/GBuffer direction.
  Do not treat it as a direct source-level material graph or classic BasePass pixel shader.

TextureBindings prove runtime t#/u# binding only.
They do not prove UE material parameter names unless separate cooked metadata also supports the mapping.

If renderdoc_svt_summary.json reports Detected=true:
  Treat SVT page-table textures, physical texture pages, and feedback/runtime UAVs as UE renderer runtime mechanisms.
  Reconstruct Unity material shaders with direct Unity texture properties instead of porting SVT indirection.

If renderdoc_cpd_summary.json reports partial:
  The CPD path may be proven even when actual per-instance CUSTOM_DATA_OFFSET values were not decoded.
  Do not fabricate CPD values. Treat decoded CPD values, when present, as runtime facts rather than UE source parameter names.

Unity target remains Unity 6 URP Deferred with DOTS instancing unless the user explicitly changes the target.
Reuse Unity URP Deferred lighting unless lightpass audit or a RenderDoc lightpass overlay proves game-specific custom lighting.
```

## Completion Criteria

The task is complete only when:

```text
1. The RenderDoc directory exists and contains drawcall.json.
2. The script command finishes without failure.
3. OUT_DIR exists.
4. renderdoc_runtime_overlay.json exists and is parseable.
5. renderdoc_runtime_overlay.md exists.
6. renderdoc_material_runtime_evidence.md/json exists.
7. renderdoc_shader_io_summary.json exists.
8. renderdoc_cpd_summary.json exists.
9. renderdoc_svt_summary.json exists.
10. renderdoc_gbuffer_outputs.json exists.
11. renderdoc_shader_match.json exists.
12. renderdoc_texture_slot_map.json exists.
13. renderdoc_runtime_outputs.json exists.
14. All JSON files in OUT_DIR parse.
15. If Bundle=... was provided or auto-detected, `--context-only "BUNDLE_DIR"` was run and returned `Verify: OK`.
16. Final response reports:
   - RenderDoc directory
   - Bundle directory, or "not provided"
   - Output directory
   - overlay Status
   - DrawcallRole RoleCandidate and SpecificRole
   - ShaderMatch summary
   - CPD status
   - SVT status
   - GBuffer output count/role
```

If any step fails, report the exact failed command, exit result, and the missing or invalid file.
