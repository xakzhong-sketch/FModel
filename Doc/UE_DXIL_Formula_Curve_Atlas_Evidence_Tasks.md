# UE DXIL Formula And Curve Atlas Evidence Tasks

Status: implemented and verified against `D:\Tmp\ShaderBundles\MI_CG_RockSmooth_01a.bundle`.

## Scope

Implement two static evidence passes for cooked UE material bundles:

- DXIL formula evidence summary: compact, selected-entrypoint evidence for Layer/Blend/Function formulas.
- Curve atlas cooked metadata summary: cooked texture/curve metadata needed to recover gradient row count and sampling assumptions.

The output must stay AI Agent friendly and must not inline large DXIL disassembly.

## Task List

### 1. Documentation Bootstrap

Status: done

- Create this task document under `Doc`.
- Keep task status updated as implementation progresses.
- Final update must include verification commands and any known limitations.

### 2. Curve Atlas Candidate Discovery

Status: done

- Scan `parameters/textures.json` for texture parameters and referenced textures containing `curve`, `gradient`, or `atlas`.
- Scan `analysis/material_layer_parameter_bindings.json` for layer/blend texture bindings and string values containing curve atlas hints.
- Scan `analysis/material_function_dependencies.json`, `source/material.cooked.json`, and `source/material_functions/*.cooked.json` for cooked references to curve/gradient/atlas assets.
- Normalize UE object paths and de-duplicate candidates by object path/name.

### 3. Curve Atlas Cooked Metadata Export

Status: done

- In full export mode, resolve candidate texture packages through CUE4Parse and export cooked JSON to `source/textures/<CurveAtlasName>.cooked.json` when not already present.
- In `--semantic-only` or `--context-only` mode, reuse existing `source/textures/*.cooked.json`; do not fail only because provider access is unavailable.
- Extract facts such as `SizeX`, `SizeY`, `PixelFormat`, `SRGB/bSRGB`, `CompressionSettings`, virtual texture flags, and curve/gradient metadata arrays when present.
- Infer row-count candidates conservatively from cooked curve arrays first, then from texture dimensions only as an assumption.

### 4. Curve Atlas Analysis Outputs

Status: done

- Write `analysis/curve_atlas_metadata.json`.
- Write `analysis/curve_atlas_metadata.md`.
- Include explicit evidence level for each row-count or sampling-rule conclusion.
- Add output files to semantic status and manifest file collection.

### 5. DXIL Formula Target Discovery

Status: done

- Read `analysis/reconstruction_entrypoints.json` and use primary selected pixel shaders before raw shader dumps.
- Discover formula targets from material layer stack, layer contracts, material function dependencies, static permutation data, and known module names such as `MB_MaskID`, `HeightLerp`, `HistogramScan`, `CheapContrast`, `ML_LayerTint`, `NormalFromHeightmap`, `NormalIntensity`, and curve/gradient modules.
- Keep targets compact and avoid generating a target for every raw parameter.

### 6. DXIL Formula Evidence Extraction

Status: done

- Parse selected `.dxil.ll` files and collect compact instruction evidence: texture samples, cbuffer loads, arithmetic operations, compare/select, saturate/min/max, normalize-like operations, and output writes.
- Detect formula patterns conservatively:
  - height/mask lerp
  - cheap contrast
  - histogram scan or band/window mask
  - normal reconstruction/intensity
  - curve/gradient atlas sampling
  - vertex/color mask overlay where evidence exists
- Record `supported`, `partial`, or `not_found` status per formula. Do not claim UE source graph recovery from anonymous registers.

### 7. DXIL Formula Analysis Outputs

Status: done

- Write `analysis/dxil_formula_evidence.json`.
- Write `analysis/dxil_formula_evidence.md`.
- Write `analysis/module_formula_evidence.json`.
- Include selected shader file references and compact line/operator evidence only.
- Add output files to semantic status and manifest file collection.

### 8. Pipeline Integration

Status: done

- Run curve atlas metadata pass from semantic analysis for full and offline refresh modes.
- Run DXIL formula evidence pass after `analysis/reconstruction_entrypoints.json` has been generated, so selected entrypoints are available.
- Ensure `--context-only` and `--semantic-only` can repair or refresh the new outputs when possible.

### 9. Agent Workspace And Skill Updates

Status: done

- Add `analysis/module_formula_evidence.json`, `analysis/dxil_formula_evidence.json/md`, and `analysis/curve_atlas_metadata.json/md` to generated read-first guidance.
- Update bundle `README.md`, `AGENTS.md`, `WORKFLOW.md`, `NEXT_TASK.md`, `PROMPT_NEXT_SESSION.md`, `agent_context.json`, and local skill text so future Agents use compact formula/curve evidence before raw DXIL.
- Preserve the existing Unity 6 URP Deferred + DOTS instancing target and `.mat` access restrictions.

### 10. Verification

Status: done

- Build `CUE4Parse.ShaderBundleExporter`.
- Refresh a representative existing bundle with `--semantic-only` and `--context-only` when available.
- Verify JSON parseability and Agent workspace integrity with `--verify-only`.
- Confirm generated docs reference the new files and still warn against broad raw DXIL loading.

## Acceptance Criteria

- New exports contain compact curve atlas metadata and DXIL formula evidence files.
- Offline refresh does not require original pak/provider access for already exported cooked JSON.
- Formula output separates proven cooked DXIL dataflow from inferred source-material intent.
- Agent docs instruct future Unity reconstruction Agents to use module formula evidence and curve atlas metadata before reading selected raw DXIL.

## Verification

Commands run from `D:\Github\FModel`:

```powershell
dotnet build CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release --nologo -v:minimal
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- --semantic-only "D:\Tmp\ShaderBundles\MI_CG_RockSmooth_01a.bundle" --verbose
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- --context-only "D:\Tmp\ShaderBundles\MI_CG_RockSmooth_01a.bundle" --verbose
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- --verify-only "D:\Tmp\ShaderBundles\MI_CG_RockSmooth_01a.bundle" --verbose
```

Observed result:

- Build succeeded. The optional native build still reports missing `cmake`, then continues without CUE4Parse-Natives, matching existing project behavior.
- `--semantic-only` refreshed `analysis/curve_atlas_metadata.json/md` and ended with `Verify: OK`.
- `--context-only` refreshed `analysis/dxil_formula_evidence.json/md`, `analysis/module_formula_evidence.json`, Agent docs, and ended with `Verify: OK`.
- `--verify-only` ended with `Verify: OK`.

## Known Limitations

- `dxil_formula_evidence` is selected-entrypoint pattern evidence, not a full UE material graph decompiler.
- Anonymous DXIL registers still require semantic maps, texture/register statistics, and optional RenderDoc evidence before promotion to named UE parameter semantics.
- Curve atlas row count is only strong when backed by cooked curve metadata. Texture-dimension-only candidates remain assumptions.
