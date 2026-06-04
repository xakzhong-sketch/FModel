# UE Cooked Shader Bundle Exporter Implementation Plan

本文档给出正式 `shader bundle exporter` 的落地计划。它基于已验证的 Subnautica 2 / `M_Character_Teeth` 流程，把当前临时代码整理成可复用、可复现、AI Agent 友好的导出工具。

关联工作流文档：

- `Doc/UE_Cooked_Shader_Bundle_Workflow.md`

## 目标

实现一个命令行 exporter：

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --game Subnautica2 `
  --paks "D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\Subnautica2\Content\Paks" `
  --mapping "D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\5.6.1-114707+++Project+SN2-Release-Hotfix-Live-Subnautica2.usmap" `
  --material "/Game/Materials/_Master/Master/M_Character_Teeth" `
  --out "D:\Tmp\ShaderBundles\M_Character_Teeth.bundle" `
  --decompress-shader "D:\Github\UEShaderMapExtractor\Build\decompress_shader.exe" `
  --dxc "C:\Program Files (x86)\Windows Kits\10\bin\10.0.26100.0\x64\dxc.exe"
```

输出 deterministic bundle：

```text
M_Character_Teeth.bundle/
  manifest.json
  source/
    material.cooked.json
    shader_archive.metadata.json
  parameters/
    material_parameters.json
    textures.json
  shaders/
    000_vs.dxil
    000_vs.dxil.ll
    000_vs.reflection.json
    001_ps.dxil
    001_ps.dxil.ll
    001_ps.reflection.json
  groups/
    group_02843.compressed
    group_02843.bin
  analysis/
    shader_map.json
    shader_entries.json
    resource_bindings.json
    inferred_roles.json
  generated/
    unity_builtin.shader
    reconstruction_notes.md
```

## 非目标

本工具不承诺：

- 还原原始 UE Material Graph。
- 还原原始变量名、材质函数名、节点拓扑。
- 自动生成与 UE 渲染管线完全一致的 Unity ShaderLab。
- 把 UE BasePass / Lumen / GBuffer / Nanite 上下文无损搬到 Unity。

本工具承诺：

- 原始 cooked shader bytecode 精确保留。
- bytecode 到 disassembly/reflection/analysis 的生成过程可复现。
- 事实和推断分离，方便 AI Agent 后续处理。

## 推荐代码位置

不要继续把正式逻辑堆在 `CUE4Parse.Example`。

建议新增独立 console project：

```text
CUE4Parse/
  CUE4Parse.ShaderBundleExporter/
    CUE4Parse.ShaderBundleExporter.csproj
    Program.cs
    Exporter/
      ShaderBundleExportOptions.cs
      ShaderBundleExporter.cs
      MaterialShaderMapResolver.cs
      ShaderArchiveResolver.cs
      IoStoreShaderGroupExtractor.cs
      ShaderContainerExtractor.cs
      DxcDisassembler.cs
      DxcReflectionParser.cs
      MaterialParameterExtractor.cs
      BundleManifestWriter.cs
      UnityShaderApproxGenerator.cs
```

`CUE4Parse.ShaderBundleExporter` 引用：

- `CUE4Parse`
- `Newtonsoft.Json`
- `Serilog`，可选

## CLI 设计

必填参数：

```text
--game <name>
--paks <path>
--mapping <path>
--material <ue-object-path-or-vfs-path>
--out <bundle-output-dir>
```

建议参数：

```text
--shader-archive <path-or-name>       可选；不填时自动选择 project archive
--decompress-shader <exe-path>        可选；不填时仅导出 compressed group，不做解压
--dxc <exe-path>                      可选；不填时跳过 disasm/reflection
--overwrite                           允许覆盖已有 bundle
--keep-groups                         保留 groups/*.compressed 和 groups/*.bin
--no-generated-unity                  不生成 Unity 近似 shader
--verbose                             输出详细日志
```

首版只需要支持：

```text
--game Subnautica2
--paks
--mapping
--material
--out
--decompress-shader
--dxc
--overwrite
```

## 模块职责

### ShaderBundleExportOptions

负责 CLI 参数结构化：

```csharp
public sealed class ShaderBundleExportOptions
{
    public string Game { get; init; }
    public string PaksDirectory { get; init; }
    public string MappingPath { get; init; }
    public string MaterialPath { get; init; }
    public string OutputDirectory { get; init; }
    public string? ShaderArchive { get; init; }
    public string? DecompressShaderPath { get; init; }
    public string? DxcPath { get; init; }
    public bool Overwrite { get; init; }
    public bool KeepGroups { get; init; }
    public bool GenerateUnityApproximation { get; init; }
}
```

### ShaderBundleExporter

主编排器。负责：

1. 初始化 `DefaultFileProvider`。
2. 解析 material。
3. 定位 shader map。
4. 定位 shader archive。
5. 提取 group chunk。
6. 解压 group。
7. 切 shader container。
8. 运行 `dxc -dumpbin`。
9. 生成 JSON 和可选 Unity 近似版。

### MaterialShaderMapResolver

输入：

```text
DefaultFileProvider
material path
```

输出：

```json
{
  "materialObjectPath": "/Game/Materials/_Master/Master/M_Character_Teeth",
  "materialName": "M_Character_Teeth",
  "shaderPlatform": "SP_PCD3D_SM6",
  "resourceHash": "B6Vt7r3Zar4gqd1oPg2ppGwN0VY=",
  "hasInlineCode": false,
  "loadedMaterialResourceIndex": 0
}
```

实现细节：

- 支持 `/Game/.../AssetName` 和 `Subnautica2/Content/.../AssetName.uasset` 两种输入。
- 查找 `UMaterialInterface` export。
- 读取 `LoadedMaterialResources[i].LoadedShaderMap`。
- 若 `LoadedMaterialResources` 为空，给出明确诊断：
  - mapping 是否正确。
  - game profile 是否正确。
  - `ReadShaderMaps` 是否开启。

### ShaderArchiveResolver

输入：

```text
DefaultFileProvider
shader platform
optional --shader-archive
```

输出：

```text
GameFile projectShaderArchive
FShaderCodeArchive metadata
FIoStoreShaderCodeArchive ioArchive
```

选择规则：

1. 如果指定 `--shader-archive`，优先使用指定 archive。
2. 否则查找包含 game/project name 的 `ShaderArchive-*-PCD3D_SM6-PCD3D_SM6.ushaderbytecode`。
3. 优先 project archive，后备 global archive。
4. 如果 `ResourceHash` 只在 global archive 中找到，也应支持。

### IoStoreShaderGroupExtractor

输入：

```text
FIoStoreShaderCodeArchive
ShaderMapEntry index
Mounted IoStoreReader[]
```

输出：

```text
groups/group_XXXXX.compressed
groups/group_XXXXX.bin
```

逻辑：

```text
ResourceHash
  -> ShaderMapHashes[index]
  -> ShaderMapEntries[index]
  -> ShaderIndices range
  -> ShaderEntries[shaderIndex]
  -> ShaderGroupIndex
  -> ShaderGroupIoHashes[groupIndex]
  -> IoStoreReader.Read(chunkId)
```

若 `rawGroup.Length == ShaderGroupEntries[groupIndex].UncompressedSize`：

- 直接写 `group_XXXXX.bin`。

若不相等：

- 写 `group_XXXXX.compressed`。
- 调用 `decompress_shader.exe <compressed> <uncompressedSize>`。
- 确认生成 `.bin` 且大小匹配。

### ShaderContainerExtractor

输入：

```text
groupData
ShaderEntries in group
target shader index set
```

输出：

```text
shaders/000_vs.dxil
shaders/001_ps.dxil
shaders/002_cs.dxil
```

切片逻辑：

```text
start = current.UncompressedOffsetInGroup
end = next.UncompressedOffsetInGroup 或 groupData.Length
slice = groupData[start..end]
```

container 查找：

```text
DXBC magic = 44 58 42 43
DXIL magic = 44 58 49 4C
DXBC header size offset = magicOffset + 24
```

命名规则：

```text
{ordinal:D3}_{stageLower}_idx_{shaderIndex}_group_{groupIndex:D5}.dxil
```

阶段在 `dxc -dumpbin` 后才能确认。首轮可先输出：

```text
000_unknown_idx_46521_group_02843.dxil
```

解析 stage 后再 rename 或在 manifest 中记录。

### DxcDisassembler

输入：

```text
*.dxil
```

命令：

```powershell
dxc.exe -dumpbin shader.dxil
```

输出：

```text
shader.dxil.ll
```

失败处理：

- 保留 `.dxil`。
- 在 manifest 中记录 `disassembly.status = failed`。
- 不阻塞 bundle 基础输出。

### DxcReflectionParser

输入：

```text
shader.dxil.ll
```

输出：

```json
{
  "stage": "Pixel",
  "shaderModel": "ps_6_6",
  "shaderHash": "9b51a251beacf5ec6e482f96a9664aa3",
  "inputSignature": [],
  "outputSignature": [],
  "resourceBindings": [],
  "cbuffers": []
}
```

首版可用文本解析：

- `; Pixel Shader`
- `!dx.shaderModel = !{!"ps", i32 6, i32 6}`
- `; shader hash:`
- `; Input signature:`
- `; Output signature:`
- `; Resource Bindings:`
- `; Buffer Definitions:`

后续可改用更稳定的 DXC reflection API。

### MaterialParameterExtractor

输入：

```text
material.cooked.json
```

输出：

```text
parameters/material_parameters.json
parameters/textures.json
```

样例输出：

```json
{
  "shadingModel": "MSM_Subsurface",
  "numericParameters": [
    {"name": "SpecularBase", "type": "Scalar", "value": 0.33},
    {"name": "AO Power in Specular", "type": "Scalar", "value": 1.0},
    {"name": "Gums Scattering", "type": "Scalar", "value": 0.25},
    {"name": "Teeth Scattering", "type": "Scalar", "value": 0.6},
    {"name": "Tongue Scattering", "type": "Scalar", "value": 0.4}
  ],
  "textureParameters": [
    {"name": "BC", "textureIndex": 0, "textureName": "T_Default_BCM"},
    {"name": "NRO", "textureIndex": 1, "textureName": "T_Default_NRH"},
    {"name": "Mask", "textureIndex": 2, "textureName": "T_Default_OAE"}
  ]
}
```

### BundleManifestWriter

负责写：

```text
manifest.json
analysis/shader_map.json
analysis/shader_entries.json
analysis/resource_bindings.json
analysis/inferred_roles.json
```

要求：

- 所有路径使用相对 bundle 根目录路径。
- 所有事实和推断分开。
- 记录 extractor version、game profile、mapping 文件、原始 paks 路径。
- 记录每个步骤的成功/失败状态。

### UnityShaderApproxGenerator

可选模块。

输入：

```text
parameters/material_parameters.json
parameters/textures.json
analysis/inferred_roles.json
```

输出：

```text
generated/unity_builtin.shader
generated/reconstruction_notes.md
```

原则：

- 只生成近似版。
- 文件顶部注释写清楚不是 bytecode 等价翻译。
- 引用 `manifest.json` 中的 `resourceHash` 和 shader stage 统计。

## 实施阶段

### Phase 0: 清理临时代码

目标：

- 保留当前 `CUE4Parse.Example` 的验证逻辑作为参考。
- 不继续扩展 Example。
- 新增独立 exporter project。

验收：

- `dotnet build CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release` 成功。

### Phase 1: CLI 和 provider 初始化

实现：

- CLI 参数解析。
- `EGame` 解析：`Subnautica2 -> EGame.GAME_Subnautica2`。
- 初始化 Oodle/Zlib。
- 初始化 `DefaultFileProvider`。
- mount/post-mount。

验收：

```text
Mounted files > 0
Mounted VFS > 0
```

错误诊断：

- paks 路径不存在。
- mapping 文件不存在。
- game name 不支持。
- VFS mount 失败。

### Phase 2: material 解析和 ResourceHash 导出

实现：

- material path normalization。
- `provider.LoadPackage(materialFile)`。
- 导出 `source/material.cooked.json`。
- 提取 `LoadedShaderMap.ResourceHash`。
- 提取 material parameters/textures。

验收样例：

```text
ResourceHash = B6Vt7r3Zar4gqd1oPg2ppGwN0VY=
ShadingModel = MSM_Subsurface
Texture params = BC/NRO/Mask
```

### Phase 3: shader archive metadata 解析

实现：

- 自动定位 project shader archive。
- 读取 `FShaderCodeArchive`。
- 写 `source/shader_archive.metadata.json`。
- 从 `ShaderMapHashes` 定位 `ResourceHash`。

验收样例：

```text
Shader map index = 2680
NumShaders = 14
```

### Phase 4: IoStore shader group 提取和解压

实现：

- 收集目标 `ShaderGroupIndex`。
- 通过 `ShaderGroupIoHashes` 读取 IoStore chunk。
- 写 `groups/*.compressed`。
- 调用 `decompress_shader.exe`。
- 验证 `.bin` 大小。

验收样例：

```text
Target shader groups = 6
groups/*.bin count = 6
group bin sizes match ShaderGroupEntries.UncompressedSize
```

### Phase 5: shader container 切片

实现：

- 按 `UncompressedOffsetInGroup` 切片。
- 检测 DXBC/DXIL container。
- 写 `shaders/*.dxil` 或 `shaders/*.dxbc`。

验收样例：

```text
shader containers = 14
all containers are .dxil for PCD3D_SM6
```

### Phase 6: dxc disasm 和 reflection JSON

实现：

- 对每个 `.dxil` 调用 `dxc -dumpbin`。
- 写 `.dxil.ll`。
- 解析 stage/hash/resource bindings/signatures。
- 写 `.reflection.json`。

验收样例：

```text
5 Vertex Shader
6 Pixel Shader
3 Compute Shader
```

### Phase 7: bundle manifest 和 analysis JSON

实现：

- 写 `manifest.json`。
- 写 `analysis/shader_map.json`。
- 写 `analysis/shader_entries.json`。
- 写 `analysis/resource_bindings.json`。
- 写 `analysis/inferred_roles.json`。

验收：

- bundle 中所有文件都能从 manifest 相对路径找到。
- `facts` 不含推断字段。
- `inferred` 字段带 `confidence`。

### Phase 8: Unity 近似版生成

实现：

- 根据材质参数生成 Built-in Surface Shader 近似版。
- 可选增加 URP Unlit/Lit 版本。
- 写 reconstruction notes。

验收：

- `.shader` 可导入 Unity。
- 参数名对应 `parameters/material_parameters.json`。
- notes 明确说明非 bytecode 等价。

### Phase 9: 批量处理支持

首版单材质完成后，再扩展：

```text
--materials-file materials.txt
--materials-glob "Subnautica2/Content/Materials/**/*.uasset"
--out-root D:\Tmp\ShaderBundles
```

批量模式要求：

- 单材质失败不终止全局流程。
- 每个 bundle 有独立 log。
- 输出 summary report。

## 测试计划

### 手动 golden case

输入：

```text
/Game/Materials/_Master/Master/M_Character_Teeth
```

期望：

```text
ResourceHash = B6Vt7r3Zar4gqd1oPg2ppGwN0VY=
ShaderMapIndex = 2680
TargetShaders = 14
TargetGroups = 6
DXIL files = 14
Stage summary = 5 VS / 6 PS / 3 CS
```

### 自动检查

实现一个 `--verify-only` 或导出后自动校验：

- `manifest.json` 可反序列化。
- 所有 manifest 路径存在。
- 每个 `.dxil` 非空。
- 每个 `.dxil.ll` 中能解析 stage。
- resource binding 数量非负。
- material parameters JSON 非空。

### 回归样例

至少准备：

- 一个 shader archive material：`M_Character_Teeth`。
- 一个没有 shader archive 的 material，如果后续找到。
- 一个 GlobalShader-only case，如果需要支持 post-process/global shader。

## 风险和处理

### UEShaderMapExtractor 原脚本不支持 UE5 IoStore grouped archive

风险：

- `extractShaderFromArchive.py` 期望 `ShaderEntries.Offset/Size/UncompressedSize`。
- UE5 IoStore grouped archive 使用 `Packed/ShaderGroupIndex/UncompressedOffsetInGroup`。

处理：

- 不依赖原脚本。
- 只复用 `decompress_shader.exe`。
- group payload 由 CUE4Parse `IoStoreReader.Read(FIoChunkId)` 读取。

### ShaderType hash 无法映射名称

风险：

- AI Agent 不知道 shader entry 具体是 `TBasePassPS` 还是其他 permutation。

处理：

- Phase 1-8 先用 stage/resource binding 分析。
- 后续增加 `.stinfo` 解析，把 hash 映射到 shader type / vertex factory type。

### Unity 结果和 UE 不一致

风险：

- UE 的 GBuffer/Lumen/Subsurface/Nanite 上下文无法直接在 Unity 中复刻。

处理：

- Unity 版本标记为 approximation。
- 精确保真层始终是 `.dxil + .ll + reflection.json`。

### DXC 路径不可用

处理：

- CLI 允许 `--dxc`。
- 未提供时自动搜索 Windows Kits 常见路径。
- 找不到时跳过 disasm，但保留 `.dxil`。

### decompress_shader.exe 不可用

处理：

- CLI 允许 `--decompress-shader`。
- 未提供时只导出 compressed group。
- manifest 记录 group 解压失败。
- 后续可考虑把 Oodle/LZ4/Zlib/Zstd 解压逻辑内置到 C#。

## 交付物

首版交付：

```text
CUE4Parse/CUE4Parse.ShaderBundleExporter/
Doc/UE_Cooked_Shader_Bundle_Exporter_Implementation_Plan.md
```

首版命令能完整导出：

```text
D:\Tmp\ShaderBundles\M_Character_Teeth.bundle
```

并满足：

```text
14 DXIL
14 DXIL disasm
14 reflection JSON
material parameters JSON
textures JSON
manifest JSON
Unity URP approximation shader （unity6）
```

## 建议执行顺序

1. 新建 `CUE4Parse.ShaderBundleExporter` console project。
2. 从当前 `CUE4Parse.Example/Program.cs` 提取已验证逻辑。
3. 先硬跑通 `M_Character_Teeth`，保证结果和当前临时输出一致。
4. 再抽象 CLI、模块和 JSON schema。
5. 最后补 Unity approximation 和 batch mode。

