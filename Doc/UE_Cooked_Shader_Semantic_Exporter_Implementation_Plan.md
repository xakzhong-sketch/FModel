# UE Cooked Shader Semantic Exporter Implementation Plan

本文档定义 `CUE4Parse.ShaderBundleExporter` 的下一阶段实现计划：从“DXIL 提取器”升级为“语义证据生成器”，为后续 Unity 6 URP Deferred + DOTS instancing shader 还原提供结构化输入。

目标不是保证源级 material graph 100% 还原，而是最大化可验证事实、降低 AI Agent 猜测比例，并把所有推断都标注证据和置信度。

## Current Baseline

当前 exporter 已能输出：

```text
manifest.json
source/material.cooked.json
source/shader_archive.metadata.json
parameters/material_parameters.json
parameters/textures.json
groups/*.compressed
groups/*.bin
shaders/*.dxil
shaders/*.dxil.ll
shaders/*.reflection.json
analysis/shader_map.json
analysis/shader_entries.json
analysis/resource_bindings.json
analysis/inferred_roles.json
analysis/shader_type_info.json
```

当前已知缺口：

```text
.stinfo readable shader type / vertex factory / permutation names unsupported
Material Function dependencies are listed but not exported as cooked JSON
DXIL resource usage is not traced deeply enough to map registers to semantics
Texture asset metadata is not exported
UE5.6 GBuffer / MRT output semantics are not mapped
Virtual texture / feedback UAV / renderer-only mechanisms are not classified
Unity Deferred reconstruction has no structured contract input
```

## Design Principles

1. 所有输出分为 `facts`、`candidates`、`inferred`。
2. 任何非确定映射都必须有 `confidence` 和 `evidence`。
3. 不直接生成 Unity shader；生成 Unity Agent 可消费的 contract。
4. 不把 UE runtime renderer feature 当成可直接移植的 material shader 逻辑。
5. 每个新 analyzer 都写独立 JSON，最后汇总到 `analysis/unity_deferred_reconstruction_contract.json`。

## Target Output Layout

新增输出：

```text
source/material_functions/*.cooked.json
source/textures/*.cooked.json
source/shader_type_info/*.stinfo

analysis/material_function_dependencies.json
analysis/texture_assets.json
analysis/shader_type_info.json
analysis/dxil_resource_usage.json
analysis/uniform_buffer_usage.json
analysis/texture_register_candidates.json
analysis/texture_channel_semantics.json
analysis/gbuffer_semantics.json
analysis/runtime_features.json
analysis/semantic_binding_map.json
analysis/unity_deferred_reconstruction_contract.json
```

`manifest.json` 需要新增：

```json
{
  "SemanticAnalysis": {
    "Status": "complete|partial|failed",
    "Files": [],
    "Unsupported": [],
    "Warnings": []
  }
}
```

## Phase 1 - FunctionInfos Dependency Export

### Goal

自动导出 target material 引用的 Material Function cooked JSON，并建立依赖图。

### Inputs

```text
source/material.cooked.json
parameters/material_parameters.json
provider.Files
FunctionInfos
```

### Outputs

```text
source/material_functions/MF_FadeCharacterNearCamera.cooked.json
source/material_functions/MF_Normal_RedGreenChannel_Recombine.cooked.json
source/material_functions/MF_NormalIntensity.cooked.json
analysis/material_function_dependencies.json
```

### Implementation

新增模块：

```text
Exporter/MaterialFunctionDependencyExporter.cs
Models/MaterialFunctionDependencyDocument.cs
```

步骤：

1. 从 `MaterialParametersDocument.FunctionInfos` 提取 `ObjectPath`。
2. 将 `/Game/Materials/Functions/MF_Name.0` 规范化为 package path：

```text
Content/Materials/Functions/MF_Name.uasset
```

3. 在 `provider.Files` 中定位 `.uasset`。
4. `provider.LoadPackage(functionFile)`。
5. `JsonConvert.SerializeObject(exports, Formatting.Indented)`。
6. 写入 `source/material_functions/{AssetName}.cooked.json`。
7. 递归扫描 function JSON 中的 nested `FunctionInfos`。
8. 防止环形依赖：用 visited package path set。
9. 写 dependency graph。

### JSON Schema

```json
{
  "Material": "/Game/Materials/_Master/Master/M_Character_Teeth",
  "Functions": [
    {
      "Name": "MF_NormalIntensity",
      "ObjectPath": "/Game/Materials/Functions/MF_NormalIntensity.0",
      "PackagePath": "Subnautica2/Content/Materials/Functions/MF_NormalIntensity.uasset",
      "CookedJson": "source/material_functions/MF_NormalIntensity.cooked.json",
      "Depth": 0,
      "Dependencies": [],
      "Status": "exported"
    }
  ],
  "Missing": [],
  "Cycles": []
}
```

### Acceptance

Golden case 至少导出：

```text
MF_FadeCharacterNearCamera
MF_Normal_RedGreenChannel_Recombine
MF_NormalIntensity
```

并且 `analysis/material_function_dependencies.json` 能列出每个 function 的 cooked JSON 相对路径。

## Phase 2 - Texture Asset Metadata Exporter

### Goal

导出材质引用贴图的 cooked JSON 和可用于语义判断的 texture metadata。

### Inputs

```text
parameters/textures.json
provider.Files
```

### Outputs

```text
source/textures/T_Default_BCM.cooked.json
source/textures/T_Default_NRH.cooked.json
source/textures/T_Default_OAE.cooked.json
analysis/texture_assets.json
```

### Implementation

新增模块：

```text
Exporter/TextureAssetMetadataExporter.cs
Models/TextureAssetMetadataDocument.cs
```

步骤：

1. 从 `ReferencedTextures` 和 `TextureParameters` 收集 texture object path。
2. 解析 `/Game/.../T_Name.0` 为 `.uasset`。
3. 导出 texture package cooked JSON。
4. 提取 metadata：
   - `TextureName`
   - `ObjectPath`
   - `PixelFormat`
   - `CompressionSettings`
   - `SRGB`
   - `LODBias`
   - `SizeX`
   - `SizeY`
   - `VirtualTextureStreaming`
   - `NeverStream`
   - `SamplerSource`
5. 如果 CUE4Parse 类型字段不稳定，则保留 raw JSON path，并用 JSON token 递归搜索常见字段。

### JSON Schema

```json
{
  "Textures": [
    {
      "ParameterName": "NRO",
      "TextureName": "T_Default_NRH",
      "CookedJson": "source/textures/T_Default_NRH.cooked.json",
      "Facts": {
        "SRGB": false,
        "CompressionSettings": "TC_Normalmap",
        "PixelFormat": "PF_BC5"
      },
      "SemanticHints": [
        {
          "Semantic": "normal_map_candidate",
          "Confidence": 0.8,
          "Evidence": ["parameter name NRO", "normal compression or BC5"]
        }
      ]
    }
  ]
}
```

### Acceptance

Golden case 输出 BC / NRO / Mask 三个 texture metadata，并至少能记录每张贴图的 cooked JSON 路径。

## Phase 3 - `.stinfo` Parser

### Goal

把 shader type / vertex factory / permutation role 从 hash/index 提升为可读信息。

### Inputs

```text
ShaderTypeInfo-Global-PCD3D_SM6-PCD3D_SM6.stinfo
ShaderTypeInfo-Subnautica2-PCD3D_SM6-PCD3D_SM6.stinfo
analysis/shader_entries.json
source/material.cooked.json
```

### Outputs

```text
source/shader_type_info/*.stinfo
analysis/shader_type_info.json
analysis/shader_entries.json
```

### Implementation

新增模块：

```text
Exporter/ShaderTypeInfoExporter.cs
Exporter/ShaderTypeInfoParser.cs
Models/ShaderTypeInfoDocument.cs
```

步骤：

1. 自动定位 `.stinfo`：

```text
ShaderTypeInfo-Global-PCD3D_SM6-PCD3D_SM6.stinfo
ShaderTypeInfo-Subnautica2-PCD3D_SM6-PCD3D_SM6.stinfo
```

2. 将原始 `.stinfo` 写到 `source/shader_type_info/`。
3. 研究格式：
   - 先用二进制 header / string table 探测。
   - 搜索 ASCII / UTF-16 shader type names。
   - 对照 UE `FShaderMapPointerTable.Types`、`VFTypes` hash。
4. 解析出：
   - shader type hash -> name
   - vertex factory hash -> name
   - permutation id/name if available
   - source filename if available
5. 更新 `analysis/shader_entries.json`：

```json
{
  "ShaderIndex": 46521,
  "Stage": "Pixel Shader",
  "ShaderTypeHash": "...",
  "ShaderTypeName": "TBasePassPSF...",
  "VertexFactoryTypeHash": "...",
  "VertexFactoryTypeName": "FLocalVertexFactory",
  "Permutation": "...",
  "Role": "BasePassPixel",
  "Confidence": 0.9
}
```

### Fallback

如果 parser 不能完全解析：

```json
{
  "Status": "partial",
  "ReadableStringsExtracted": [],
  "HashMappings": [],
  "UnsupportedSections": []
}
```

不能回退成现在的纯 `unsupported`，至少要保留原始 `.stinfo` 和 strings scan 结果。

### Acceptance

Golden case 至少为部分 shader 输出：

```text
shaderTypeName
vertexFactoryTypeName or candidate
role candidate
```

如果没有稳定解析，必须输出 `partial` 和 `evidence`。

## Phase 4 - DXIL Resource / Channel Usage Analyzer

### Goal

从 DXIL disassembly 中追踪 resource handle、cbuffer load、texture sample、UAV write、MRT output，形成寄存器使用证据。

### Inputs

```text
shaders/*.dxil.ll
shaders/*.reflection.json
analysis/resource_bindings.json
analysis/shader_entries.json
```

### Outputs

```text
analysis/dxil_resource_usage.json
analysis/uniform_buffer_usage.json
analysis/texture_register_candidates.json
```

### Implementation

新增模块：

```text
Exporter/DxilResourceUsageAnalyzer.cs
Exporter/DxilLlParser.cs
Models/DxilResourceUsageDocument.cs
```

核心解析目标：

```text
dx.op.createHandle
dx.op.annotateHandle
dx.op.cbufferLoadLegacy
dx.op.sample
dx.op.sampleLevel
dx.op.textureLoad
dx.op.storeOutput
dx.op.rawBufferLoad
dx.op.rawBufferStore
dx.op.bufferUpdateCounter
```

步骤：

1. 建立 SSA value -> operation map。
2. 解析 `%handle = call ... createHandle(... rangeId, index, nonUniform)`。
3. 将 handle 关联到 reflection resource binding：

```text
t0 / t1 / cb0 / u0 / spaceN
```

4. 解析 sample/load 操作：
   - texture register
   - sampler register
   - coordinate source
   - sampled component usage
5. 解析 `extractvalue` / `fadd` / `fmul` / `storeOutput` 的近邻数据流。
6. 解析 cbuffer load：
   - cbuffer register
   - register index
   - component usage
   - value consumers
7. 解析 UAV write：
   - UAV register
   - write operation
   - feedback/runtime feature candidate
8. 解析 MRT output：
   - `SV_Target0..N`
   - component source categories

### JSON Schema

```json
{
  "Shaders": [
    {
      "ShaderFile": "shaders/006_unknown_idx_46517_group_02843.dxil",
      "Stage": "Pixel Shader",
      "Resources": [
        {
          "Register": "t0",
          "Type": "texture",
          "Operations": ["sample", "sampleLevel"],
          "ChannelsUsed": ["r", "g", "b", "a"],
          "CandidateParameter": "BC",
          "Confidence": 0.65,
          "Evidence": ["binding order candidate", "sampled rgb feeds SV_Target0"]
        }
      ],
      "CBufferLoads": [
        {
          "Register": "cb2",
          "LoadIndex": 5,
          "Components": ["x"],
          "ConsumerOps": ["fmul", "fadd"],
          "CandidateField": null,
          "Confidence": 0.0
        }
      ],
      "Outputs": [
        {
          "Semantic": "SV_Target0",
          "ComponentsWritten": ["x", "y", "z", "w"],
          "CandidateMeaning": "GBuffer0/BaseColorCandidate",
          "Confidence": 0.5
        }
      ]
    }
  ]
}
```

### Acceptance

Golden case 至少输出：

```text
per shader resource usage
per shader cbuffer load indexes
per shader texture channel usage
per pixel shader MRT output count
UAV write detection if present
```

## Phase 5 - Texture Register Candidate Mapper

### Goal

把匿名 `t0/t1/t2` 映射到 `BC/NRO/Mask` 等 texture parameters，输出候选而非硬判定。

### Inputs

```text
parameters/textures.json
analysis/texture_assets.json
analysis/dxil_resource_usage.json
source/material.cooked.json
```

### Outputs

```text
analysis/texture_register_candidates.json
analysis/texture_channel_semantics.json
```

### Scoring Model

证据项：

```text
UniformTextureParameters order
ReferencedTextures index
texture parameter name
texture asset metadata
DXIL channel usage
GBuffer output consumer path
known UE packing conventions
```

示例评分：

```json
{
  "Register": "t1",
  "Candidates": [
    {
      "ParameterName": "NRO",
      "TextureName": "T_Default_NRH",
      "Confidence": 0.78,
      "Evidence": [
        "UniformTextureParameters TextureIndex = 1",
        "parameter name NRO suggests normal/roughness/occlusion packing",
        "sampled channels feed normal-like math"
      ]
    }
  ]
}
```

### Channel Semantics

```json
{
  "TextureParameter": "Mask",
  "Channels": {
    "r": {
      "Semantic": "subsurface_or_region_mask_candidate",
      "Confidence": 0.4,
      "Evidence": ["used near scattering scalar math"]
    },
    "g": {
      "Semantic": "occlusion_candidate",
      "Confidence": 0.55,
      "Evidence": ["parameter AO Power in Specular exists", "feeds GBuffer-like scalar"]
    }
  }
}
```

### Acceptance

Golden case 输出 BC / NRO / Mask 的 register candidates；如果无法确定，必须说明原因。

## Phase 6 - UE5.6 GBuffer Schema Mapper

### Goal

将 pixel shader MRT 输出与 UE5.6 deferred GBuffer 语义建立候选映射，帮助 Unity Deferred 还原判断 BaseColor、Normal、Roughness、Specular、AO、ShadingModel 等。

### Inputs

```text
analysis/dxil_resource_usage.json
shaders/*.reflection.json
UE5.6 PCD3D_SM6 GBuffer layout knowledge
```

### Outputs

```text
analysis/gbuffer_semantics.json
```

### Implementation

新增模块：

```text
Exporter/UeGBufferSchemaMapper.cs
Models/GBufferSemanticsDocument.cs
```

步骤：

1. 对每个 pixel shader 统计 `SV_TargetN` 数量。
2. 按 UE5.6 deferred base pass 常见 layout 建立候选：

```text
SV_Target0 -> GBufferA candidate
SV_Target1 -> GBufferB candidate
SV_Target2 -> GBufferC candidate
SV_Target3 -> GBufferD/custom data candidate
SV_Target4 -> velocity/custom/depth-related candidate
```

3. 结合写入值的数据来源：
   - texture sample rgb
   - normal decode math
   - scalar cbuffer
   - constants
   - shading model id
4. 输出候选语义和置信度。

### Important Constraint

UE GBuffer layout 受项目 renderer settings、Substrate、platform defines、permutation 影响，不能无证据硬编码。所有映射都是 candidates。

### JSON Schema

```json
{
  "Platform": "PCD3D_SM6",
  "Engine": "UE5.6",
  "Shaders": [
    {
      "ShaderFile": "...dxil",
      "MrtCount": 5,
      "Targets": [
        {
          "Target": "SV_Target0",
          "CandidateSemantic": "BaseColor_or_GBufferA",
          "Confidence": 0.55,
          "Evidence": ["rgb path includes BC sample"]
        }
      ]
    }
  ]
}
```

### Acceptance

Golden case 输出每个 pixel shader 的 MRT 数量和 `SV_TargetN` candidate semantics。

## Phase 7 - Runtime Feature Classifier

### Goal

识别不适合直接移植到 Unity material shader 的 UE runtime renderer features。

### Inputs

```text
analysis/dxil_resource_usage.json
analysis/resource_bindings.json
shaders/*.dxil.ll
```

### Outputs

```text
analysis/runtime_features.json
```

### Features To Detect

```text
virtual texture page-table read
virtual texture physical texture indirection
feedback UAV write
scene texture reads
custom primitive data
instance data buffer
nanite / virtual shadow / renderer-only buffers
compute-only passes
```

### Classification

```json
{
  "Features": [
    {
      "Feature": "VirtualTextureFeedbackUAV",
      "Shaders": ["...dxil"],
      "Registers": ["u0"],
      "UnityHandling": "abstract",
      "Reason": "Requires UE virtual texture streaming/runtime feedback system",
      "RecommendedUnityStrategy": "Use regular Texture2D inputs for material reconstruction; document missing runtime VT feedback."
    }
  ]
}
```

### Acceptance

Golden case 如果存在 VT / UAV 行为，必须标注为 renderer runtime feature，而不是要求 Unity shader 直接移植。

## Phase 8 - Semantic Binding Map

### Goal

汇总 `.stinfo`、DXIL usage、texture metadata、GBuffer mapping、runtime classifier，生成一个 AI Agent 可直接消费的语义映射。

### Inputs

```text
analysis/shader_type_info.json
analysis/dxil_resource_usage.json
analysis/texture_assets.json
analysis/texture_register_candidates.json
analysis/texture_channel_semantics.json
analysis/gbuffer_semantics.json
analysis/runtime_features.json
parameters/material_parameters.json
parameters/textures.json
```

### Outputs

```text
analysis/semantic_binding_map.json
```

### JSON Schema

```json
{
  "Material": "M_Character_Teeth",
  "TextureBindings": [
    {
      "ParameterName": "BC",
      "TextureName": "T_Default_BCM",
      "CandidateRegisters": [
        {
          "Register": "t0",
          "Shaders": ["..."],
          "Confidence": 0.72,
          "Evidence": []
        }
      ],
      "ChannelSemantics": {
        "rgb": {
          "Semantic": "base_color_candidate",
          "Confidence": 0.65
        },
        "a": {
          "Semantic": "alpha_or_mask_candidate",
          "Confidence": 0.35
        }
      }
    }
  ],
  "UniformBindings": [
    {
      "ParameterName": "SpecularBase",
      "Value": 0.33,
      "CandidateCBufferLoads": [],
      "Confidence": 0.3,
      "Evidence": ["material parameter exists", "no exact cbuffer field name available"]
    }
  ],
  "RuntimeFeatures": [],
  "KnownUnknowns": []
}
```

### Acceptance

Unity Agent 可以只读此文件和 contract 文件，就知道哪些 mapping 是事实、哪些是候选。

## Phase 9 - Unity Deferred Reconstruction Contract

### Goal

为 Unity 6 URP Deferred + DOTS instancing 还原生成最终结构化输入。

### Outputs

```text
analysis/unity_deferred_reconstruction_contract.json
```

### Contract Schema

```json
{
  "Target": {
    "UnityVersion": "Unity 6",
    "RenderPipeline": "URP",
    "RenderingPath": "Deferred",
    "DotsInstancing": true
  },
  "Material": {
    "Name": "M_Character_Teeth",
    "OriginalPath": "/Game/Materials/_Master/Master/M_Character_Teeth",
    "ShadingModel": {
      "UE": "MSM_Subsurface",
      "UnityStrategy": "Deferred approximation",
      "Confidence": 0.45,
      "Evidence": []
    }
  },
  "Properties": [
    {
      "UnityName": "_BaseMap",
      "UEParameter": "BC",
      "Type": "Texture2D",
      "TextureName": "T_Default_BCM",
      "Semantic": "base_color_candidate",
      "Confidence": 0.65,
      "Evidence": []
    }
  ],
  "DeferredOutputs": {
    "Strategy": "URP GBuffer/deferred compatible approximation",
    "GBufferCandidates": []
  },
  "RuntimeFeaturesToAbstract": [],
  "RequiredDocumentation": [
    "List exact facts",
    "List inferred mappings",
    "List unsupported UE runtime features",
    "List compile validation status"
  ],
  "BlockedIf": [
    "Cannot identify a URP Deferred-compatible pass structure in target Unity project",
    "DOTS instancing cannot be represented or compile-tested"
  ]
}
```

### Acceptance

Contract 必须：

```text
包含 BC/NRO/Mask texture slots
包含 scalar parameters
包含 MSM_Subsurface strategy
包含 runtime feature handling
包含 confidence/evidence
明确 Unity target = Unity 6 URP Deferred
明确 DOTS instancing required
```

## Phase 10 - CLI And Verification

### New CLI Options

```text
--semantic-analysis
--export-dependencies
--export-texture-assets
--parse-stinfo
--analyze-dxil-usage
--write-unity-contract
--semantic-only <existing bundle>
```

默认行为建议：

```text
single export 默认启用 semantic-analysis
verify-only 检查 semantic files if present
semantic-only 对已有 bundle 补跑语义分析
```

### Verify Updates

`--verify-only` 新增检查：

```text
analysis/material_function_dependencies.json can parse
analysis/texture_assets.json can parse
analysis/dxil_resource_usage.json can parse
analysis/semantic_binding_map.json can parse
analysis/unity_deferred_reconstruction_contract.json can parse
all confidence values are 0..1
all evidence arrays exist for inferred/candidate fields
```

## Implementation Order

推荐顺序：

```text
Task 01 - FunctionInfos dependency export
Task 02 - Texture asset metadata export
Task 03 - Preserve and scan .stinfo files
Task 04 - Implement partial .stinfo parser
Task 05 - DXIL resource usage parser
Task 06 - Uniform buffer usage output
Task 07 - Texture register candidate mapper
Task 08 - Texture channel semantic mapper
Task 09 - Runtime feature classifier
Task 10 - UE5.6 GBuffer schema mapper
Task 11 - Semantic binding map aggregator
Task 12 - Unity Deferred reconstruction contract
Task 13 - CLI flags and semantic-only mode
Task 14 - Verify updates
Task 15 - Golden case regeneration under D:\ShaderWP
Task 16 - Update AGENTS.md / Skill docs to consume the contract
```

## Risk Register

| Risk | Impact | Mitigation |
|---|---|---|
| `.stinfo` format differs across UE versions/games | readable names incomplete | output partial parser result and raw strings scan |
| DXIL optimized SSA loses source structure | graph-level reconstruction impossible | trace resource usage and values only as evidence |
| cbuffer names stripped | no exact field names | correlate with material uniform parameters and offsets, keep candidates |
| texture register order ambiguous | wrong texture mapping | multi-evidence scoring with confidence |
| UE GBuffer layout varies | wrong MRT semantics | emit candidates, not facts |
| VT/UAV runtime logic not portable | Unity shader incomplete | classify as runtime feature to abstract |
| Unity Deferred implementation depends on URP package internals | compile risk | require target Unity project/package inspection before final shader |

## Definition Of Done

Golden case `M_Character_Teeth` must produce:

```text
source/material_functions/*.cooked.json count >= 3
analysis/material_function_dependencies.json
source/textures/*.cooked.json count >= 3
analysis/texture_assets.json
analysis/shader_type_info.json status != unsupported
analysis/dxil_resource_usage.json
analysis/uniform_buffer_usage.json
analysis/texture_register_candidates.json
analysis/texture_channel_semantics.json
analysis/gbuffer_semantics.json
analysis/runtime_features.json
analysis/semantic_binding_map.json
analysis/unity_deferred_reconstruction_contract.json
```

`D:\ShaderWP\commands\verify_bundle.ps1` must pass after regeneration.

The Unity Agent should be able to start from:

```text
analysis/unity_deferred_reconstruction_contract.json
analysis/semantic_binding_map.json
```

and only open DXIL/disassembly files for deeper evidence checks.
