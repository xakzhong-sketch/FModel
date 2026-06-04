# UE Cooked Shader Semantic Exporter Tasks

本文档把 `UE_Cooked_Shader_Semantic_Exporter_Implementation_Plan.md` 拆成可执行 task。

目标：把 `CUE4Parse.ShaderBundleExporter` 从“DXIL 提取器”升级为“语义证据生成器”，为 Unity 6 URP Deferred + DOTS instancing shader 还原提供结构化 contract。

默认 golden case：

```text
Game: Subnautica2
Material: /Game/Materials/_Master/Master/M_Character_Teeth
Bundle: D:\ShaderWP\M_Character_Teeth.bundle
Expected base shaders: 14
Expected groups: 6
Expected stages: 5 VS / 6 PS / 3 CS
Expected material functions:
  MF_FadeCharacterNearCamera
  MF_Normal_RedGreenChannel_Recombine
  MF_NormalIntensity
Expected textures:
  BC -> T_Default_BCM
  NRO -> T_Default_NRH
  Mask -> T_Default_OAE
Unity target: Unity 6 URP Deferred + DOTS instancing
```

## Task Status Legend

```text
PENDING      未开始
IN_PROGRESS 进行中
DONE         已完成并通过验收
BLOCKED      被外部条件阻塞
PARTIAL      已有可用输出，但没有达到完整目标
```

## Task 00 - Semantic Analysis Architecture

Status: `DONE`

Depends on: existing `CUE4Parse.ShaderBundleExporter`

Goal:

建立 semantic analysis 的代码组织、共享模型、manifest 扩展和 feature flags。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/Semantic/
CUE4Parse/CUE4Parse.ShaderBundleExporter/Models/SemanticModels.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/SemanticAnalysisPipeline.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/BundleManifestWriter.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/ShaderBundleExportOptions.cs
```

Outputs:

```text
manifest.json
analysis/semantic_status.json
```

Steps:

1. 新增 semantic pipeline 入口。
2. 新增 `SemanticAnalysisStatus` model。
3. 在 manifest 中加入 `SemanticAnalysis` section。
4. 所有 semantic 输出路径使用 bundle-relative path。
5. 所有 candidate/inferred 字段统一支持：

```json
{
  "Confidence": 0.0,
  "Evidence": [],
  "SourceFiles": []
}
```

6. 默认 single export 启用 semantic analysis。
7. `--verify-only` 对 semantic 输出做宽松校验：存在则解析，不存在不影响旧 bundle。

Acceptance:

```text
dotnet build succeeds
Golden export still produces original exact bundle files
manifest.json contains SemanticAnalysis
analysis/semantic_status.json exists
```

## Task 01 - CLI Flags For Semantic Passes

Status: `DONE`

Depends on: Task 00

Goal:

添加可控 CLI flags，支持完整导出和对已有 bundle 补跑 semantic pass。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/ShaderBundleExportOptions.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Program.cs
```

New CLI:

```text
--semantic-analysis
--no-semantic-analysis
--export-dependencies
--export-texture-assets
--parse-stinfo
--analyze-dxil-usage
--write-unity-contract
--semantic-only <existing bundle>
```

Steps:

1. 扩展 options model。
2. 更新 help 文案。
3. `--semantic-only` 读取已有 `manifest.json`，不重新提取 DXIL。
4. `--no-semantic-analysis` 禁用所有新增语义 pass。
5. 子 flags 可单独开启对应 pass。
6. 默认行为：single export 自动启用 `--semantic-analysis`。

Acceptance:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- --help
```

Help 中能看到所有 semantic flags。

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- --semantic-only D:\ShaderWP\M_Character_Teeth.bundle
```

能在已有 bundle 上补跑 semantic pass。

## Task 02 - FunctionInfos Dependency Export

Status: `DONE`

Depends on: Task 00

Goal:

自动导出 target material 引用的 Material Function `.uasset` cooked JSON，并生成依赖图。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/Semantic/MaterialFunctionDependencyExporter.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Models/MaterialFunctionDependencyDocument.cs
```

Inputs:

```text
parameters/material_parameters.json
source/material.cooked.json
provider.Files
```

Outputs:

```text
source/material_functions/*.cooked.json
analysis/material_function_dependencies.json
```

Steps:

1. 从 `FunctionInfos` 读取 function `ObjectName` / `ObjectPath`。
2. 把 `/Game/.../MF_Name.0` 解析为 `.uasset` package path。
3. 在 `provider.Files` 中定位 function package。
4. `provider.LoadPackage(functionFile)`。
5. 写 cooked JSON 到 `source/material_functions/{Name}.cooked.json`。
6. 递归扫描 function JSON 中的 nested `FunctionInfos`。
7. 用 visited set 防止循环依赖。
8. 输出 missing/cycle/error 列表。

Acceptance:

Golden case 至少输出：

```text
source/material_functions/MF_FadeCharacterNearCamera.cooked.json
source/material_functions/MF_Normal_RedGreenChannel_Recombine.cooked.json
source/material_functions/MF_NormalIntensity.cooked.json
analysis/material_function_dependencies.json
```

Dependency JSON 中每个 function 有：

```text
Name
ObjectPath
PackagePath
CookedJson
Depth
Status
Dependencies
```

## Task 03 - Texture Asset Metadata Export

Status: `DONE`

Depends on: Task 00

Goal:

导出材质引用贴图的 cooked JSON 和 metadata，为 texture semantic 判断提供证据。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/Semantic/TextureAssetMetadataExporter.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Models/TextureAssetMetadataDocument.cs
```

Inputs:

```text
parameters/textures.json
provider.Files
```

Outputs:

```text
source/textures/*.cooked.json
analysis/texture_assets.json
```

Steps:

1. 收集 `ReferencedTextures` 和 `TextureParameters`。
2. 将 texture object path 解析为 `.uasset` package path。
3. 导出 texture package cooked JSON。
4. 从 CUE4Parse export object 或 JSON token 中提取：

```text
TextureName
ObjectPath
PixelFormat
CompressionSettings
SRGB
SizeX
SizeY
LODBias
VirtualTextureStreaming
NeverStream
SamplerSource
```

5. 生成 `SemanticHints`：

```text
normal_map_candidate
base_color_candidate
mask_packed_candidate
linear_data_candidate
srgb_color_candidate
```

Acceptance:

Golden case 输出：

```text
source/textures/T_Default_BCM.cooked.json
source/textures/T_Default_NRH.cooked.json
source/textures/T_Default_OAE.cooked.json
analysis/texture_assets.json
```

每个 texture 至少有 cooked JSON path 和 parameter name。

## Task 04 - Preserve And Scan `.stinfo`

Status: `DONE`

Depends on: Task 00

Goal:

即使 parser 不完整，也先保存 `.stinfo` 原始文件并扫描可读 strings。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/Semantic/ShaderTypeInfoExporter.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/Semantic/StInfoStringScanner.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Models/ShaderTypeInfoDocument.cs
```

Inputs:

```text
provider.Files
ShaderTypeInfo-*.stinfo
```

Outputs:

```text
source/shader_type_info/*.stinfo
analysis/shader_type_info.json
```

Steps:

1. 定位 `.stinfo` 文件。
2. 写原始 `.stinfo` 到 `source/shader_type_info/`。
3. 扫描 ASCII / UTF-16 strings。
4. 识别疑似 shader type / vertex factory / source filename strings。
5. 输出 `Status = partial`，而不是 `unsupported`。
6. 记录 parser 未支持的 sections。

Acceptance:

Golden case 输出：

```text
source/shader_type_info/ShaderTypeInfo-Global-PCD3D_SM6-PCD3D_SM6.stinfo
source/shader_type_info/ShaderTypeInfo-Subnautica2-PCD3D_SM6-PCD3D_SM6.stinfo
analysis/shader_type_info.json
```

`analysis/shader_type_info.json` 中：

```text
Status != unsupported
CandidateFiles count >= 2
ReadableStrings exists
```

## Task 05 - `.stinfo` Hash Mapping Parser

Status: `DONE`

Depends on: Task 04

Goal:

解析 `.stinfo` 中的 shader type / vertex factory / permutation 映射。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/Semantic/ShaderTypeInfoParser.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Models/ShaderTypeInfoDocument.cs
```

Inputs:

```text
source/shader_type_info/*.stinfo
source/material.cooked.json
analysis/shader_entries.json
```

Outputs:

```text
analysis/shader_type_info.json
analysis/shader_entries.json
```

Steps:

1. 研究 `.stinfo` header 和 string table。
2. 建立 hash -> readable name mapping。
3. 对照 material cooked JSON 中 `FShader.Type.Hash` / `VFType.Hash`。
4. 将 mapping 写入 shader entries：

```text
shaderTypeName
vertexFactoryTypeName
permutationName
sourceFilename
roleCandidate
confidence
evidence
```

5. 如果无法完全解析，保留 partial mapping 和 raw string evidence。

Acceptance:

Golden case 至少为部分 shader entries 输出：

```text
shaderTypeName or shaderTypeNameCandidate
vertexFactoryTypeName or vertexFactoryTypeNameCandidate
roleCandidate
confidence
evidence
```

## Task 06 - DXIL LL Parser Core

Status: `DONE`

Depends on: Task 00

Goal:

建立可复用的 DXIL `.ll` 半结构化 parser，用于后续 resource usage 分析。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/Semantic/DxilLlParser.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Models/DxilInstructionModels.cs
```

Inputs:

```text
shaders/*.dxil.ll
```

Outputs:

```text
internal parsed instruction graph
```

Steps:

1. 解析 SSA assignment：

```text
%23 = call ...
%24 = extractvalue ...
%25 = fmul ...
```

2. 记录 op name、result id、argument ids、literal constants。
3. 解析 comments 中的 DXIL opcode names。
4. 建立 `%id -> instruction` map。
5. 支持 basic reverse-use 查询：value -> consumers。

Acceptance:

对 golden case 任意 `.dxil.ll`：

```text
能解析 createHandle / cbufferLoadLegacy / sample / storeOutput 行
能输出 instruction count
能查询某个 value 的 consumers
```

## Task 07 - DXIL Resource Usage Analyzer

Status: `DONE`

Depends on: Task 06

Goal:

追踪 resource handle、texture sample/load、UAV write、MRT output。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/Semantic/DxilResourceUsageAnalyzer.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Models/DxilResourceUsageDocument.cs
```

Inputs:

```text
shaders/*.dxil.ll
shaders/*.reflection.json
analysis/resource_bindings.json
analysis/shader_entries.json
```

Outputs:

```text
analysis/dxil_resource_usage.json
```

Steps:

1. 解析 `dx.op.createHandle`。
2. 将 handle 关联到 reflection 中的 register：

```text
t0
t1
cb0
u0
spaceN
```

3. 解析 texture operations：

```text
dx.op.sample
dx.op.sampleLevel
dx.op.textureLoad
```

4. 记录 texture register、sampler、channels used。
5. 解析 UAV write operations。
6. 解析 `dx.op.storeOutput`，记录 `SV_TargetN` 写入。
7. 输出 per-shader resource usage。

Acceptance:

Golden case 输出：

```text
analysis/dxil_resource_usage.json
per shader resources
per shader texture operations
per shader output targets
UAV usage if present
```

## Task 08 - Uniform Buffer Usage Analyzer

Status: `DONE`

Depends on: Task 06, Task 07

Goal:

提取 cbuffer load 证据，为 UE 参数和 Unity uniform 映射提供候选。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/Semantic/UniformBufferUsageAnalyzer.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Models/UniformBufferUsageDocument.cs
```

Inputs:

```text
analysis/dxil_resource_usage.json
parameters/material_parameters.json
shaders/*.dxil.ll
```

Outputs:

```text
analysis/uniform_buffer_usage.json
```

Steps:

1. 解析 `dx.op.cbufferLoadLegacy.*`。
2. 记录：

```text
shader
cbuffer register
load index
component
value type
consumer ops
nearby constants
```

3. 尝试与 material scalar/vector 参数做候选关联。
4. 所有字段名映射必须是 candidate，不可硬判。

Acceptance:

Golden case 输出：

```text
all pixel shader cbuffer load indexes
all vertex shader cbuffer load indexes
candidate material parameter links if evidence exists
```

## Task 09 - Texture Register Candidate Mapper

Status: `DONE`

Depends on: Task 03, Task 07

Goal:

把匿名 texture register 映射到 UE texture parameters 的候选列表。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/Semantic/TextureRegisterCandidateMapper.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Models/TextureRegisterCandidateDocument.cs
```

Inputs:

```text
parameters/textures.json
analysis/texture_assets.json
analysis/dxil_resource_usage.json
source/material.cooked.json
```

Outputs:

```text
analysis/texture_register_candidates.json
```

Scoring evidence:

```text
UniformTextureParameters order
ReferencedTextures TextureIndex
parameter name
texture asset metadata
DXIL sample channel usage
GBuffer output path
```

Steps:

1. 为每个 shader register 生成 texture parameter candidates。
2. 每个 candidate 有 confidence。
3. 每个 candidate 有 evidence。
4. 输出 unresolved registers。

Acceptance:

Golden case 对 BC / NRO / Mask 均输出 candidate register mapping。

## Task 10 - Texture Channel Semantic Mapper

Status: `DONE`

Depends on: Task 03, Task 07, Task 09

Goal:

推断 texture channel 语义，例如 base color、normal、roughness、AO、mask。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/Semantic/TextureChannelSemanticMapper.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Models/TextureChannelSemanticDocument.cs
```

Inputs:

```text
analysis/texture_register_candidates.json
analysis/texture_assets.json
analysis/dxil_resource_usage.json
parameters/material_parameters.json
```

Outputs:

```text
analysis/texture_channel_semantics.json
```

Steps:

1. 为每个 texture parameter 生成 channel map。
2. 识别 normal-like channel usage。
3. 识别 roughness/specular/AO-like scalar usage。
4. 识别 region/scattering mask candidate。
5. 所有非确定语义输出 confidence/evidence。

Acceptance:

Golden case 输出：

```text
BC.rgb base_color_candidate
NRO channel semantics candidates
Mask channel semantics candidates
unknown channels explicitly listed
```

## Task 11 - Runtime Feature Classifier

Status: `DONE`

Depends on: Task 07

Goal:

识别 UE runtime renderer-only features，避免 Unity Agent 误以为必须直接移植。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/Semantic/RuntimeFeatureClassifier.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Models/RuntimeFeatureDocument.cs
```

Inputs:

```text
analysis/dxil_resource_usage.json
analysis/resource_bindings.json
shaders/*.dxil.ll
```

Outputs:

```text
analysis/runtime_features.json
```

Features:

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

Steps:

1. 根据 UAV writes 检测 feedback/runtime feature。
2. 根据 resource names / binding patterns / DXIL ops 检测 virtual texture。
3. 根据 shader role/stage 检测 compute-only pass。
4. 对每个 feature 输出 Unity handling strategy：

```text
abstract
approximate
requires renderer feature
unsupported
```

Acceptance:

Golden case 如果发现 VT/UAV，必须输出：

```text
Feature
Shaders
Registers
UnityHandling
Reason
RecommendedUnityStrategy
```

## Task 12 - UE5.6 GBuffer Schema Mapper

Status: `DONE`

Depends on: Task 07, Task 10

Goal:

将 UE pixel shader MRT 输出映射成 UE5.6 GBuffer candidate semantics。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/Semantic/UeGBufferSchemaMapper.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Models/GBufferSemanticsDocument.cs
```

Inputs:

```text
analysis/dxil_resource_usage.json
analysis/texture_channel_semantics.json
shaders/*.reflection.json
```

Outputs:

```text
analysis/gbuffer_semantics.json
```

Steps:

1. 对每个 pixel shader 统计 `SV_TargetN`。
2. 记录 MRT count。
3. 结合 data source category：

```text
texture sample
cbuffer scalar
constant
normal decode
shading model id
```

4. 输出 `SV_TargetN -> candidate semantic`。
5. 明确 GBuffer layout 受 UE settings/permutation 影响，所有映射均为 candidate。

Acceptance:

Golden case 输出：

```text
all pixel shaders
MRT count per pixel shader
SV_TargetN candidate semantics
confidence/evidence
```

## Task 13 - Semantic Binding Map Aggregator

Status: `DONE`

Depends on: Task 02-12

Goal:

汇总所有语义证据，生成 AI Agent 可消费的统一 binding map。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/Semantic/SemanticBindingMapAggregator.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Models/SemanticBindingMapDocument.cs
```

Inputs:

```text
analysis/material_function_dependencies.json
analysis/texture_assets.json
analysis/shader_type_info.json
analysis/dxil_resource_usage.json
analysis/uniform_buffer_usage.json
analysis/texture_register_candidates.json
analysis/texture_channel_semantics.json
analysis/gbuffer_semantics.json
analysis/runtime_features.json
parameters/material_parameters.json
parameters/textures.json
```

Outputs:

```text
analysis/semantic_binding_map.json
```

Steps:

1. 合并 texture bindings。
2. 合并 scalar/vector uniform candidates。
3. 合并 shader role candidates。
4. 合并 runtime feature handling。
5. 输出 `KnownUnknowns`。
6. 每个 candidate 都有 confidence/evidence。

Acceptance:

Golden case 输出：

```text
TextureBindings for BC/NRO/Mask
UniformBindings for SpecularBase/scattering params
RuntimeFeatures section
KnownUnknowns section
```

## Task 14 - Unity Deferred Reconstruction Contract

Status: `DONE`

Depends on: Task 13

Goal:

生成 Unity Agent 的最终结构化输入，目标为 Unity 6 URP Deferred + DOTS instancing。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/Semantic/UnityDeferredContractWriter.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Models/UnityDeferredReconstructionContract.cs
```

Inputs:

```text
analysis/semantic_binding_map.json
analysis/gbuffer_semantics.json
analysis/runtime_features.json
parameters/material_parameters.json
parameters/textures.json
```

Outputs:

```text
analysis/unity_deferred_reconstruction_contract.json
```

Steps:

1. 写 Target：

```text
UnityVersion = Unity 6
RenderPipeline = URP
RenderingPath = Deferred
DotsInstancing = true
```

2. 写 material info。
3. 写 properties contract：

```text
Texture2D properties
scalar/vector properties
default values
source UE parameter
semantic candidate
confidence/evidence
```

4. 写 Deferred strategy。
5. 写 runtime features to abstract。
6. 写 required documentation/checklist。
7. 写 blocked-if 条件。

Acceptance:

Contract JSON 包含：

```text
BC/NRO/Mask
SpecularBase
Gums Scattering
Teeth Scattering
Tongue Scattering
MSM_Subsurface strategy
RuntimeFeaturesToAbstract
Unity target = Unity 6 URP Deferred
DotsInstancing = true
```

## Task 15 - Verify Semantic Outputs

Status: `DONE`

Depends on: Task 02-14

Goal:

扩展 `--verify-only`，校验 semantic outputs 的结构完整性。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/BundleVerifier.cs
```

Checks:

```text
analysis/material_function_dependencies.json parseable
analysis/texture_assets.json parseable
analysis/shader_type_info.json parseable
analysis/dxil_resource_usage.json parseable
analysis/uniform_buffer_usage.json parseable
analysis/texture_register_candidates.json parseable
analysis/texture_channel_semantics.json parseable
analysis/gbuffer_semantics.json parseable
analysis/runtime_features.json parseable
analysis/semantic_binding_map.json parseable
analysis/unity_deferred_reconstruction_contract.json parseable
all confidence values are 0..1
candidate/inferred fields have evidence arrays
manifest SemanticAnalysis file paths exist
```

Acceptance:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- --verify-only D:\ShaderWP\M_Character_Teeth.bundle
```

返回 `Verify: OK`。

## Task 16 - Golden Case Regeneration

Status: `DONE`

Depends on: Task 15

Goal:

用完整 semantic exporter 重新生成 `D:\ShaderWP\M_Character_Teeth.bundle`。

Command:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --game Subnautica2 `
  --paks "D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\Subnautica2\Content\Paks" `
  --mapping "D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\5.6.1-114707+++Project+SN2-Release-Hotfix-Live-Subnautica2.usmap" `
  --material "/Game/Materials/_Master/Master/M_Character_Teeth" `
  --out "D:\ShaderWP\M_Character_Teeth.bundle" `
  --decompress-shader "D:\Github\UEShaderMapExtractor\Build\decompress_shader.exe" `
  --dxc "C:\Program Files (x86)\Windows Kits\10\bin\10.0.26100.0\x64\dxc.exe" `
  --overwrite `
  --semantic-analysis
```

Acceptance:

```text
Base bundle exact files still pass previous verification
source/material_functions/*.cooked.json count >= 3
source/textures/*.cooked.json count >= 3
source/shader_type_info/*.stinfo count >= 2
analysis/material_function_dependencies.json exists
analysis/texture_assets.json exists
analysis/shader_type_info.json status != unsupported
analysis/dxil_resource_usage.json exists
analysis/uniform_buffer_usage.json exists
analysis/texture_register_candidates.json exists
analysis/texture_channel_semantics.json exists
analysis/gbuffer_semantics.json exists
analysis/runtime_features.json exists
analysis/semantic_binding_map.json exists
analysis/unity_deferred_reconstruction_contract.json exists
verify_bundle.ps1 returns Verify: OK
```

## Task 17 - Update ShaderWP Agent Docs

Status: `DONE`

Depends on: Task 16

Goal:

更新 `D:\ShaderWP` 的 AI Agent 文档，让后续 Unity Agent 优先消费 semantic contract。

Files:

```text
D:\ShaderWP\README.md
D:\ShaderWP\AGENTS.md
D:\ShaderWP\WORKFLOW.md
D:\ShaderWP\NEXT_TASK.md
D:\ShaderWP\PROMPT_NEXT_SESSION.md
D:\ShaderWP\agent_context.json
D:\ShaderWP\skills\unity6-urp-shader-reconstruction\SKILL.md
```

Steps:

1. 增加 `analysis/unity_deferred_reconstruction_contract.json` 为首要输入。
2. 增加 `analysis/semantic_binding_map.json` 为第二输入。
3. 明确 DXIL/disasm 只作为 evidence drill-down。
4. 保持目标：Unity 6 URP Deferred + DOTS instancing。
5. 明确 VT/UAV/runtime feature 必须参考 `runtime_features.json`。

Acceptance:

新会话只读 `PROMPT_NEXT_SESSION.md`，即可知道：

```text
先 verify bundle
先读 unity_deferred_reconstruction_contract.json
再读 semantic_binding_map.json
输出 Unity6URP files
目标 Deferred + DOTS instancing
```

## Completion Evidence

Status as of 2026-06-03: Task 00-17 已在 `CUE4Parse/CUE4Parse.ShaderBundleExporter` 实现并通过 golden case 验证。

Verified commands:

```powershell
dotnet build CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release

dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- --verify-only D:\ShaderWP\M_Character_Teeth.bundle --verbose

D:\ShaderWP\commands\verify_bundle.ps1
```

Observed results:

```text
dotnet build: Build succeeded
verify-only: Verify: OK
verify_bundle.ps1: Verify: OK
Stage summary: 6 Pixel Shader / 3 Compute Shader / 5 Vertex Shader
source/material_functions/*.cooked.json: 5
source/textures/*.cooked.json: 3
source/shader_type_info/*.stinfo: 2
analysis/semantic_status.json Status: success
analysis/shader_type_info.json Status: partial
analysis/dxil_resource_usage.json Shaders: 14
analysis/dxil_resource_usage.json TextureOps: 55
analysis/dxil_resource_usage.json CBufferLoads: 356
analysis/dxil_resource_usage.json UavOps: 18
analysis/unity_deferred_reconstruction_contract.json Target: Unity 6 URP Deferred, DOTS instancing true
```

## Suggested Execution Order

```text
Task 00
Task 01
Task 02
Task 03
Task 04
Task 05
Task 06
Task 07
Task 08
Task 09
Task 10
Task 11
Task 12
Task 13
Task 14
Task 15
Task 16
Task 17
```

## Minimal Useful Milestone

如果不能一次完成全部任务，最小有用里程碑是：

```text
Task 02 - FunctionInfos dependency export
Task 03 - Texture asset metadata export
Task 04 - Preserve and scan .stinfo
Task 07 - DXIL resource usage analyzer
Task 09 - Texture register candidate mapper
Task 11 - Runtime feature classifier
Task 14 - Unity Deferred reconstruction contract
```

完成后 Unity Agent 至少能得到：

```text
material function context
texture metadata evidence
partial shader readable context
resource/register evidence
runtime feature warnings
Unity Deferred structured contract
```
