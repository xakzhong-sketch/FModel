# UE Cooked Shader Bundle Exporter Tasks

本文档把 `UE_Cooked_Shader_Bundle_Exporter_Implementation_Plan.md` 拆成可执行 task。

默认 golden case：

```text
Game: Subnautica2
Paks: D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\Subnautica2\Content\Paks
Mapping: D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\5.6.1-114707+++Project+SN2-Release-Hotfix-Live-Subnautica2.usmap
Material: /Game/Materials/_Master/Master/M_Character_Teeth
ResourceHash: B6Vt7r3Zar4gqd1oPg2ppGwN0VY=
Expected shaders: 14
Expected groups: 6
Expected stages: 5 VS / 6 PS / 3 CS
```

## Implementation Snapshot

Status as of 2026-06-03: Task 00-16 已在 `CUE4Parse/CUE4Parse.ShaderBundleExporter` 实现。

Verified:

```text
dotnet build CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- --help
Golden export: 14 DXIL, 14 DXC dumps, 14 reflection JSON, 6 groups, stage summary 5 VS / 6 PS / 3 CS
Unity shader generation is disabled; no generated Unity shader is emitted.
--verify-only success on golden bundle
--verify-only failure on temporary missing DXIL
Batch mode: 2 success / 1 failed / summary.json written
```

Known limitation:

```text
Task 12 deliberately disables Unity shader generation.
A future Unity output must be generic Unity 6 URP Deferred and support DOTS instancing before it is re-enabled.
Task 15 writes analysis/shader_type_info.json with status = unsupported.
Shader type / vertex factory hashes are preserved, but .stinfo readable-name parsing is not implemented yet.
This follows the non-blocking acceptance path for Task 15.
```

## Task Status Legend

```text
PENDING      未开始
IN_PROGRESS 进行中
DONE         已完成并通过验收
BLOCKED      被外部条件阻塞
```

## Task 00 - Create Exporter Project

Status: `DONE`

Depends on: none

Goal:

新增独立 console project，不再继续扩展 `CUE4Parse.Example`。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/
CUE4Parse/CUE4Parse.ShaderBundleExporter/CUE4Parse.ShaderBundleExporter.csproj
CUE4Parse/CUE4Parse.ShaderBundleExporter/Program.cs
```

Steps:

1. 创建 `CUE4Parse.ShaderBundleExporter` console project。
2. 引用 `CUE4Parse` project。
3. 引用 `Newtonsoft.Json`。
4. 添加最小 `Program.cs`，打印版本和 `--help`。
5. 将 project 加入 solution，如果当前 repo 有统一 solution 维护规则。

Acceptance:

```powershell
dotnet build CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release
```

命令成功，且没有破坏 `CUE4Parse.Example`。

## Task 01 - Implement CLI Options

Status: `DONE`

Depends on: Task 00

Goal:

实现首版 CLI 参数解析。

Required options:

```text
--game
--paks
--mapping
--material
--out
```

Optional options:

```text
--shader-archive
--decompress-shader
--dxc
--overwrite
--keep-groups
--verbose
```

Deprecated compatibility flag:

```text
--no-generated-unity
```

Unity shader generation is disabled in this exporter revision, so this flag is accepted as a no-op for old scripts.

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/ShaderBundleExportOptions.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Program.cs
```

Steps:

1. 定义 `ShaderBundleExportOptions`。
2. 实现简单 CLI parser。
3. 实现 `--help`。
4. 参数缺失时输出明确错误。
5. 校验 `--paks` 和 `--mapping` 路径存在。
6. 如果 `--out` 存在且未指定 `--overwrite`，停止并提示。

Acceptance:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- --help
```

能输出帮助。

缺少必填参数时返回非 0 exit code，并列出缺少的参数。

## Task 02 - Initialize CUE4Parse Provider

Status: `DONE`

Depends on: Task 01

Goal:

根据 CLI 参数初始化 CUE4Parse provider，并完成 VFS mount。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/ProviderFactory.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/GameProfileResolver.cs
```

Steps:

1. 实现 `GameProfileResolver`。
2. 首版支持 `Subnautica2 -> EGame.GAME_Subnautica2`。
3. 初始化 `OodleHelper.Initialize()`。
4. 初始化 `ZlibHelper.Initialize()`。
5. 创建 `DefaultFileProvider`。
6. 设置 `MappingsContainer`。
7. 设置 `ReadShaderMaps = true`。
8. 执行 `Initialize()`、`Mount()`、`PostMount()`。
9. 输出 mounted VFS 数量和文件数量。

Acceptance:

Golden case 下输出类似：

```text
Game version: GAME_Subnautica2
Registered VFS: 3
Mounted VFS: 2
Mounted files: 39565
```

## Task 03 - Resolve Material Package And ShaderMap

Status: `DONE`

Depends on: Task 02

Goal:

输入 material path，自动定位 package、读取 `UMaterialInterface`，提取 `ResourceHash`。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/MaterialShaderMapResolver.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Models/MaterialShaderMapInfo.cs
```

Steps:

1. 支持 `/Game/.../AssetName` 输入。
2. 支持 `Subnautica2/Content/.../AssetName.uasset` 输入。
3. 在 `provider.Files` 中定位 `.uasset`。
4. `provider.LoadPackage(materialFile)`。
5. 获取 `UMaterialInterface` exports。
6. 找到 `LoadedMaterialResources`。
7. 读取 `LoadedShaderMap.ResourceHash`。
8. 读取 `ShaderPlatform`。
9. 写出 `source/material.cooked.json`。

Acceptance:

Golden case：

```text
Material: M_Character_Teeth
LoadedMaterialResources: 1
ResourceHash: B6Vt7r3Zar4gqd1oPg2ppGwN0VY=
ShaderPlatform: SP_PCD3D_SM6
HasCode: false
```

如果 `LoadedMaterialResources` 为空，错误信息必须包含：

```text
Check game profile, mapping file, and ReadShaderMaps.
```

## Task 04 - Extract Material Parameters And Textures

Status: `DONE`

Depends on: Task 03

Goal:

从 cooked material JSON / export 对象中提取 AI 分析和后续渲染器重建需要的材质参数。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/MaterialParameterExtractor.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Models/MaterialParametersDocument.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Models/TextureParametersDocument.cs
```

Outputs:

```text
parameters/material_parameters.json
parameters/textures.json
```

Steps:

1. 提取 `ShadingModel` / `ShadingModels`。
2. 提取 `UniformNumericParameters`。
3. 提取 `UniformTextureParameters`。
4. 提取 `ReferencedTextures`。
5. 提取 `FunctionInfos`。
6. 提取 `PropertyConnectedMask`。
7. 将事实写入 JSON。
8. 不在此阶段做推断。

Acceptance:

Golden case `material_parameters.json` 包含：

```text
SpecularBase = 0.33
AO Power in Specular = 1.0
Gums Scattering = 0.25
Teeth Scattering = 0.6
Tongue Scattering = 0.4
ShadingModel = MSM_Subsurface
```

`textures.json` 包含：

```text
BC -> T_Default_BCM
NRO -> T_Default_NRH
Mask -> T_Default_OAE
```

## Task 05 - Resolve Shader Archive Metadata

Status: `DONE`

Depends on: Task 03

Goal:

自动找到 shader archive，读取 metadata，并用 `ResourceHash` 定位目标 ShaderMapEntry。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/ShaderArchiveResolver.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Models/ShaderArchiveInfo.cs
```

Outputs:

```text
source/shader_archive.metadata.json
analysis/shader_map.json
```

Steps:

1. 查找 `.ushaderbytecode` / `.ushadercode`。
2. 如果指定 `--shader-archive`，优先使用指定文件。
3. 自动优先选择 project archive。
4. fallback 到 global archive。
5. `new FShaderCodeArchive(archiveReader)`。
6. 判断是否为 `FIoStoreShaderCodeArchive`。
7. 在 `ShaderMapHashes` 中查找 `ResourceHash`。
8. 提取 `ShaderMapEntries[index]`。
9. 提取目标 `ShaderIndices`。

Acceptance:

Golden case：

```text
ShaderArchive = ShaderArchive-Subnautica2-PCD3D_SM6-PCD3D_SM6.ushaderbytecode
ShaderMapIndex = 2680
TargetShaders = 14
```

## Task 06 - Extract IoStore Shader Groups

Status: `DONE`

Depends on: Task 05

Goal:

根据目标 shader entries 的 `ShaderGroupIndex`，从 IoStore 中读取 group chunk。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/IoStoreShaderGroupExtractor.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Models/ShaderGroupInfo.cs
```

Outputs:

```text
groups/group_XXXXX.compressed
```

Steps:

1. 从目标 shader indices 收集 distinct `ShaderGroupIndex`。
2. 对每个 group 读取 `ShaderGroupIoHashes[groupIndex]`。
3. 在 mounted `IoStoreReader` 中查找 `DoesChunkExist(chunkId)`。
4. `reader.Read(chunkId)`。
5. 写出 `groups/group_XXXXX.compressed`。
6. 记录 `CompressedSize` / `UncompressedSize` / actual raw size。

Acceptance:

Golden case：

```text
TargetGroups = 6
groups/group_02843.compressed exists
groups/group_04160.compressed exists
groups/group_04301.compressed exists
groups/group_04311.compressed exists
groups/group_04318.compressed exists
groups/group_04332.compressed exists
```

## Task 07 - Decompress Shader Groups

Status: `DONE`

Depends on: Task 06

Goal:

调用 `decompress_shader.exe` 或直接 passthrough，生成解压后的 group `.bin`。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/ShaderGroupDecompressor.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/External/ExternalProcessRunner.cs
```

Outputs:

```text
groups/group_XXXXX.bin
```

Steps:

1. 如果 raw group size 等于 expected uncompressed size，直接复制为 `.bin`。
2. 否则要求 `--decompress-shader` 存在。
3. 调用：

```powershell
decompress_shader.exe group_XXXXX.compressed <UncompressedSize>
```

4. 查找 `group_XXXXX.compressed.bin`。
5. 标准化重命名或复制为 `group_XXXXX.bin`。
6. 验证 `.bin` 长度等于 `UncompressedSize`。
7. 记录 stdout/stderr/exit code。

Acceptance:

Golden case：

```text
groups/*.bin count = 6
all group bin sizes match ShaderGroupEntries.UncompressedSize
```

## Task 08 - Slice Shader Containers From Groups

Status: `DONE`

Depends on: Task 07

Goal:

从解压后的 group 中切出目标 shader entry，并写出 DXIL/DXBC container。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/ShaderContainerExtractor.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Models/ShaderContainerInfo.cs
```

Outputs:

```text
shaders/000_unknown_idx_46521_group_02843.dxil
```

Steps:

1. 对 group 内 shader entries 按 `UncompressedOffsetInGroup` 排序。
2. 对目标 shader index 计算 slice：

```text
start = current.UncompressedOffsetInGroup
end = next.UncompressedOffsetInGroup 或 groupData.Length
```

3. 保存 raw slice `.bin`，可选。
4. 查找 `DXBC` magic。
5. 判断 container 内是否含 `DXIL` magic。
6. 根据 header size 写出 `.dxil` 或 `.dxbc`。
7. 记录 slice offset、slice size、container offset、container size。

Acceptance:

Golden case：

```text
shaders/*.dxil count = 14
no zero-byte shader files
all shader containers have DXBC magic
all shader containers contain DXIL magic
```

## Task 09 - Run DXC Disassembly

Status: `DONE`

Depends on: Task 08

Goal:

对每个 DXIL 执行 `dxc -dumpbin`，生成 `.ll`。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/DxcDisassembler.cs
```

Outputs:

```text
shaders/000_unknown_idx_46521_group_02843.dxil.ll
```

Steps:

1. 如果 CLI 指定 `--dxc`，使用该路径。
2. 未指定时尝试搜索 Windows Kits 常见路径。
3. 对每个 `.dxil` 执行：

```powershell
dxc.exe -dumpbin shader.dxil
```

4. stdout 写入 `.ll`。
5. 失败时记录错误，不删除 `.dxil`。

Acceptance:

Golden case：

```text
*.dxil.ll count = 14
each .ll contains one of:
  ; Vertex Shader
  ; Pixel Shader
  ; Compute Shader
```

## Task 10 - Parse Reflection From DXC Dump

Status: `DONE`

Depends on: Task 09

Goal:

从 `.ll` 中解析 stage、shader model、signature、resource bindings，写结构化 JSON。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/DxcReflectionParser.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Models/ShaderReflectionDocument.cs
```

Outputs:

```text
shaders/000_ps_idx_46521_group_02843.reflection.json
analysis/resource_bindings.json
```

Steps:

1. 解析 stage：

```text
; Vertex Shader
; Pixel Shader
; Compute Shader
```

2. 解析 shader hash：

```text
; shader hash:
```

3. 解析 shader model：

```text
!dx.shaderModel
```

4. 解析 input signature table。
5. 解析 output signature table。
6. 解析 resource bindings table。
7. 解析 cbuffer sizes。
8. 写 per-shader reflection JSON。
9. 根据 stage rename shader 文件或仅在 manifest 中记录 normalized name。

Acceptance:

Golden case stage summary：

```text
5 Vertex Shader
6 Pixel Shader
3 Compute Shader
```

每个 reflection JSON 至少包含：

```text
stage
shaderHash
shaderModel
inputSignature
outputSignature
resourceBindings
```

## Task 11 - Write Bundle Manifest And Analysis Indexes

Status: `DONE`

Depends on: Task 03, Task 04, Task 05, Task 08, Task 10

Goal:

生成 bundle 顶层 manifest 和分析索引。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/BundleManifestWriter.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Models/BundleManifest.cs
```

Outputs:

```text
manifest.json
analysis/shader_map.json
analysis/shader_entries.json
analysis/resource_bindings.json
analysis/inferred_roles.json
```

Steps:

1. 记录输入来源：
   - game
   - paks
   - mapping
   - material path
   - shader archive
2. 记录工具路径：
   - decompress_shader
   - dxc
3. 记录 material info。
4. 记录 shader map info。
5. 记录 group info。
6. 记录 shader container info。
7. 记录 reflection summaries。
8. 所有路径用 bundle 相对路径。
9. `facts` 和 `inferred` 分开。
10. 写导出状态和错误列表。

Acceptance:

```text
manifest.json can be deserialized
all manifest file paths exist
facts section contains no inferred fields
inferred fields include confidence when present
```

## Task 12 - Disable Non-Generic Unity Generation

Status: `DONE`

Depends on: Task 04, Task 10, Task 11

Goal:

不再输出材质特定的 Unity 近似 shader。Unity 输出只有在满足通用 Unity 6 URP Deferred 且支持 DOTS instancing 时才能重新启用。

Files:

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/ShaderBundleExporter.cs
CUE4Parse/CUE4Parse.ShaderBundleExporter/Exporter/ShaderBundleExportOptions.cs
```

Outputs:

```text
none
```

Steps:

1. 移除旧的 `UnityShaderApproxGenerator`。
2. 导出流程不再调用 Unity shader generation。
3. `--no-generated-unity` 保留为 deprecated no-op，避免旧脚本报错。
4. CLI help 明确说明 Unity shader generation disabled。
5. 保留 `parameters/*.json`、`analysis/*.json`、DXIL 和 DXIL disasm 作为未来 Unity 6 URP 生成器输入。

Acceptance:

```text
Golden export does not create generated/unity_urp.shader.
Golden export does not create generated/unity_builtin.shader.
CLI help states Unity shader generation is disabled.
Bundle precise intermediate files are unaffected.
```

## Task 13 - Golden Case End-To-End Verification

Status: `DONE`

Depends on: Task 00-12

Goal:

使用 `M_Character_Teeth` 端到端验证 exporter。

Command:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --game Subnautica2 `
  --paks "D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\Subnautica2\Content\Paks" `
  --mapping "D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\5.6.1-114707+++Project+SN2-Release-Hotfix-Live-Subnautica2.usmap" `
  --material "/Game/Materials/_Master/Master/M_Character_Teeth" `
  --out "D:\Tmp\ShaderBundles\M_Character_Teeth.bundle" `
  --decompress-shader "D:\Github\UEShaderMapExtractor\Build\decompress_shader.exe" `
  --dxc "C:\Program Files (x86)\Windows Kits\10\bin\10.0.26100.0\x64\dxc.exe" `
  --overwrite
```

Acceptance:

```text
Bundle exists
manifest.json exists
source/material.cooked.json exists
source/shader_archive.metadata.json exists
parameters/material_parameters.json exists
parameters/textures.json exists
groups/*.bin count = 6
shaders/*.dxil count = 14
shaders/*.ll count = 14
shaders/*.reflection.json count = 14
Stage summary = 5 VS / 6 PS / 3 CS
no generated Unity shader exists
```

## Task 14 - Verification Command

Status: `DONE`

Depends on: Task 11

Goal:

实现 `--verify-only` 或自动导出后验证。

CLI:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --verify-only "D:\Tmp\ShaderBundles\M_Character_Teeth.bundle"
```

Checks:

1. `manifest.json` 可解析。
2. manifest 中所有路径存在。
3. 每个 `.dxil` 非空。
4. 每个 `.ll` 能解析 stage。
5. 每个 `.reflection.json` 可解析。
6. `parameters` 文件存在。
7. expected counts 与 manifest 一致。

Acceptance:

Golden bundle verify 成功，返回 exit code 0。

人为删除一个 shader 文件后 verify 失败，返回非 0，并报告缺失路径。

## Task 15 - ShaderTypeInfo Mapping

Status: `DONE`

Depends on: Task 10

Goal:

解析 `.stinfo`，把 shader type / vertex factory hash 映射为可读名称。

Inputs:

```text
ShaderTypeInfo-Global-PCD3D_SM6-PCD3D_SM6.stinfo
ShaderTypeInfo-Subnautica2-PCD3D_SM6-PCD3D_SM6.stinfo
```

Outputs:

```text
analysis/shader_type_info.json
analysis/shader_entries.json
```

Steps:

1. 定位 `.stinfo` 文件。
2. 研究 CUE4Parse 是否已有解析类型。
3. 如果没有，新增最小 parser。
4. 将 shader type hash 映射到 readable name。
5. 将 VF type hash 映射到 readable name。
6. 更新 `inferred_roles.json`。

Acceptance:

至少能为部分 shader entries 输出：

```text
shaderTypeName
vertexFactoryTypeName
```

如果解析失败，manifest 记录 `shaderTypeInfo.status = unsupported`，不影响主流程。

Implementation note:

当前实现走非阻塞路径：定位候选 `.stinfo` / `ShaderTypeInfo` 文件，写出 `analysis/shader_type_info.json`，并在 manifest 中记录 `ShaderTypeInfo.Status = unsupported`。

## Task 16 - Batch Export Mode

Status: `DONE`

Depends on: Task 13

Goal:

支持批量导出多个 material bundle。

CLI:

```text
--materials-file materials.txt
--out-root D:\Tmp\ShaderBundles
```

Steps:

1. 读取 material path 列表。
2. 每个 material 独立 bundle。
3. 单个失败不终止全局流程。
4. 输出 summary report。
5. 支持跳过已存在 bundle。

Acceptance:

输入 3 个 material：

```text
2 success
1 failed
summary.json exists
```

Implementation note:

批量模式在存在失败项时仍会完成所有条目并写出 `summary.json`；进程返回非 0，便于脚本/CI 识别部分失败。

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
Task 13
Task 12
Task 14
Task 15
Task 16
```

Unity output is intentionally disabled. Re-enable only with a generic Unity 6 URP Deferred generator that supports DOTS instancing.
