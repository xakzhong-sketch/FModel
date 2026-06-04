# UE Cooked Shader Bundle Workflow

本文档记录从 UE 打包产物中提取 cooked shader，并输出 AI Agent 友好的分析 bundle 的推荐工作流。

当前验证样例：

- Game: Subnautica 2 Early Access `0.10.1`
- UE profile: `GAME_Subnautica2`
- Shader platform: `PCD3D_SM6`
- Material: `/Game/Materials/_Master/Master/M_Character_Teeth`
- ResourceHash: `B6Vt7r3Zar4gqd1oPg2ppGwN0VY=`

## 目标

目标不是直接从 cooked shader 还原出原始 UE Material Graph，也不是直接保证 Unity ShaderLab 和 UE 渲染结果完全一致。

推荐目标是输出一个 deterministic bundle：

- 保留原始编译 shader 字节码作为真相源。
- 输出 DXIL/DXBC disassembly 作为人类和 AI 可读表示。
- 输出结构化 reflection/analysis JSON，让 AI Agent 不需要每次重新解析 `.utoc/.ucas/.ushaderbytecode`。
- 基于 bundle 再生成 Unity/GLSL/HLSL 近似或移植版本。

精确性边界：

- `DXIL/DXBC/SPIR-V` 原始字节码：可 100% 保留 cooked variant 的实际 GPU 逻辑。
- `DXIL disasm` / `DXBC asm`：接近 lossless，适合作为分析依据。
- 低层 HLSL/GLSL 反编译：方便阅读，但不保证 100% 等价。
- Unity ShaderLab/URP/Built-in：属于移植结果，不能保证与 UE 原渲染管线完全一致。

## 推荐产物结构

示例 bundle：

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
    002_cs.dxil
    002_cs.dxil.ll
    002_cs.reflection.json
  analysis/
    shader_map.json
    resource_bindings.json
    inferred_roles.json
  generated/
    unity_builtin.shader
    unity_urp.shader
```

`manifest.json` 至少应包含：

```json
{
  "game": "Subnautica2",
  "engineProfile": "GAME_Subnautica2",
  "sourcePaks": "D:/Tmp/Subnautica.2.v.0.10.1.Early.Access/Subnautica2/Subnautica2/Content/Paks",
  "mapping": "D:/Tmp/Subnautica.2.v.0.10.1.Early.Access/Subnautica2/5.6.1-114707+++Project+SN2-Release-Hotfix-Live-Subnautica2.usmap",
  "materialPath": "/Game/Materials/_Master/Master/M_Character_Teeth",
  "resourceHash": "B6Vt7r3Zar4gqd1oPg2ppGwN0VY=",
  "shaderPlatform": "PCD3D_SM6",
  "bundleVersion": 1
}
```

事实和推断必须分开记录。示例：

```json
{
  "facts": {
    "stage": "Pixel",
    "shaderModel": "ps_6_6",
    "resources": ["t0", "t1", "t2", "s0", "s1"],
    "outputs": ["SV_Target0", "SV_Target1", "SV_Target2"]
  },
  "inferred": {
    "t1LikelyRole": "normal/roughness texture",
    "confidence": 0.72
  }
}
```

## 前置工具

已验证可用：

- FModel/CUE4Parse repo: `D:\Github\FModel`
- UEShaderMapExtractor: `D:\Github\UEShaderMapExtractor`
- `decompress_shader.exe`: `D:\Github\UEShaderMapExtractor\Build\decompress_shader.exe`
- `dxc.exe`: Windows Kits，例如 `C:\Program Files (x86)\Windows Kits\10\bin\10.0.26100.0\x64\dxc.exe`

样例游戏输入：

```text
D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\Subnautica2\Content\Paks
D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\5.6.1-114707+++Project+SN2-Release-Hotfix-Live-Subnautica2.usmap
```

注意：

- 该游戏使用 shader archive。
- FModel GUI 若使用 generic `GAME_UE5_6` 或无效 usmap，材质 JSON 可能出现 `LoadedMaterialResources: []`。
- 对该游戏应使用 `GAME_Subnautica2` 和游戏目录中的 `.usmap`。

## 工作流

### 1. 判断是否使用 shader archive

在 package index 或 FModel asset tree 中查找：

```text
ShaderArchive-*.ushaderbytecode
ShaderArchive-*.ushadercode
ShaderTypeInfo-*.stinfo
*.stable.upipelinecache
```

Subnautica 2 样例中存在：

```text
Subnautica2/Content/ShaderArchive-Global-PCD3D_SM6-PCD3D_SM6.ushaderbytecode
Subnautica2/Content/ShaderArchive-Subnautica2-PCD3D_SM6-PCD3D_SM6.ushaderbytecode
```

判断规则：

- material shader map `HasCode = true`：shader code 可能内联在 material 中。
- material shader map `HasCode = false` 且有 `ResourceHash`：shader code 在 shader archive 或 IoStore shader group 中。

### 2. 加载 material 并提取 ResourceHash

使用 CUE4Parse 载入目标 material package，要求：

- `ReadShaderMaps = true`
- `MappingsContainer = FileUsmapTypeMappingsProvider(...)`
- `VersionContainer(EGame.GAME_Subnautica2)`

样例结果：

```text
Material: M_Character_Teeth
LoadedMaterialResources: 1
Platform: SP_PCD3D_SM6
ResourceHash: B6Vt7r3Zar4gqd1oPg2ppGwN0VY=
HasCode: False
```

同时导出：

```text
source/material.cooked.json
parameters/material_parameters.json
parameters/textures.json
```

从样例 material JSON 可恢复：

```text
ShadingModel: MSM_Subsurface
Texture parameters:
  BC -> T_Default_BCM
  NRO -> T_Default_NRH
  Mask -> T_Default_OAE
Numeric parameters:
  SpecularBase = 0.33
  AO Power in Specular = 1.0
  Gums Scattering = 0.25
  Teeth Scattering = 0.6
  Tongue Scattering = 0.4
```

### 3. 读取 shader archive metadata

读取 project shader archive：

```text
Subnautica2/Content/ShaderArchive-Subnautica2-PCD3D_SM6-PCD3D_SM6.ushaderbytecode
```

CUE4Parse 类型：

```csharp
var shaderCodeArchive = new FShaderCodeArchive(archive);
var ioArchive = (FIoStoreShaderCodeArchive) shaderCodeArchive.SerializedShaders;
```

对 UE5 IoStore shader archive，`.ushaderbytecode` 主要是 metadata。真正的 shader group payload 通过 `ShaderGroupIoHashes` 存在 IoStore chunk 中。

### 4. 用 ResourceHash 定位 ShaderMapEntry

流程：

```text
ResourceHash
  -> ShaderMapHashes[index]
  -> ShaderMapEntries[index]
  -> ShaderIndices[ShaderIndicesOffset .. ShaderIndicesOffset + NumShaders)
  -> ShaderEntries[shaderIndex]
```

样例结果：

```text
Shader map index: 2680
Target shaders: 14
Target shader groups: 6
```

注意：

- UE material 对应一个 ShaderMap，不是一个 shader 文件。
- 一个 ShaderMap 会包含多个 VS/PS/CS entry，来自不同 vertex factory、base pass、lumen、compute permutation 等。

### 5. 读取 IoStore shader group chunk

对于每个目标 `FIoStoreShaderCodeEntry`：

```text
ShaderGroupIndex = ShaderEntries[shaderIndex].ShaderGroupIndex
UncompressedOffsetInGroup = ShaderEntries[shaderIndex].UncompressedOffsetInGroup
```

通过 group index 获取：

```text
ShaderGroupEntries[groupIndex]
ShaderGroupIoHashes[groupIndex]
```

使用 mounted `IoStoreReader` 读取 chunk：

```csharp
var chunkId = ioArchive.ShaderGroupIoHashes[groupIndex];
var rawGroup = ioStoreReader.Read(chunkId);
```

写出：

```text
groups/group_02843.compressed
```

如果 `rawGroup.Length != ShaderGroupEntries[groupIndex].UncompressedSize`，调用：

```powershell
D:\Github\UEShaderMapExtractor\Build\decompress_shader.exe groups\group_02843.compressed 154167
```

输出：

```text
groups/group_02843.compressed.bin
```

### 6. 从 group 中切出单个 shader

同一个 group 内的 shader 按 `UncompressedOffsetInGroup` 排序。

单 shader 范围：

```text
start = current.UncompressedOffsetInGroup
end = next.UncompressedOffsetInGroup 或 groupData.Length
size = end - start
```

写出：

```text
shaders/000_ps.bin
```

再在 `.bin` 中查找 DX container：

```text
DXBC magic: 44 58 42 43
DXIL magic: 44 58 49 4C
container size: DXBC header offset + 24
```

写出：

```text
shaders/000_ps.dxil
```

Subnautica 2 是 `PCD3D_SM6`，因此输出是 `.dxil`，不是 `.dxbc` 或 `.glsl`。

### 7. 反汇编和 reflection

使用 `dxc`：

```powershell
& 'C:\Program Files (x86)\Windows Kits\10\bin\10.0.26100.0\x64\dxc.exe' -dumpbin shaders\000_ps.dxil > shaders\000_ps.dxil.ll
```

从 `.ll` 中提取结构化信息：

- shader stage：Vertex / Pixel / Compute
- shader model：例如 `ps_6_6`
- shader hash
- input signature
- output signature
- resource bindings
- cbuffer sizes
- texture/sampler/UAV bindings
- entry point

写出：

```text
shaders/000_ps.reflection.json
analysis/resource_bindings.json
analysis/shader_entries.json
```

样例阶段统计：

```text
5 Vertex Shader
6 Pixel Shader
3 Compute Shader
```

### 8. 解析 ShaderTypeInfo

为了让 AI Agent 更好理解 shader entry 的用途，应进一步解析：

```text
ShaderTypeInfo-Global-PCD3D_SM6-PCD3D_SM6.stinfo
ShaderTypeInfo-Subnautica2-PCD3D_SM6-PCD3D_SM6.stinfo
```

目标是把 hash 映射回更可读的名称，例如：

```text
FLocalVertexFactory
TBasePassVS
TBasePassPS
FLumenCard*
FNanite*
```

没有这一步时，AI Agent 只能根据 stage、resource binding 和 IR 内容推断用途。

### 9. 生成 AI 分析 JSON

建议每个 shader entry 输出一个 `analysis` JSON：

```json
{
  "shaderId": 0,
  "stage": "Pixel",
  "shaderModel": "ps_6_6",
  "facts": {
    "inputs": ["TEXCOORD10_centroid", "TEXCOORD11_centroid", "SV_Position"],
    "outputs": ["SV_Target0", "SV_Target1", "SV_Target2"],
    "resources": [
      {"bind": "t0", "type": "texture", "format": "u32", "dimension": "2d"},
      {"bind": "t1", "type": "texture", "format": "f32", "dimension": "2d"},
      {"bind": "s0", "type": "sampler"}
    ]
  },
  "inferred": {
    "likelyPipelineRole": "UE BasePass or GBuffer pixel shader",
    "confidence": 0.6
  }
}
```

原则：

- `facts` 只能放从 bytecode/disasm/reflection/material JSON 直接读取的信息。
- `inferred` 放 AI 或规则推断的信息，并必须带置信度。

### 10. Unity/GLSL 重建

Unity shader 应作为 bundle 的下游生成物：

```text
generated/unity_builtin.shader
generated/unity_urp.shader
generated/lowlevel_hlsl.shader
generated/glsl/
```

重建策略：

- 从 `material_parameters.json` 和 `textures.json` 恢复材质参数。
- 从 Pixel Shader DXIL/LL 分析 texture sampling、mask 使用、输出语义。
- 从 ShaderTypeInfo 判断 shader entry 是否属于目标 pass。
- 生成 Unity shader 时记录哪些逻辑是精确事实，哪些是推断。

不能承诺 Unity ShaderLab 与 UE cooked shader 完全一致，因为 UE 的 View/Primitive/Material/Lumen/GBuffer/Nanite 上下文无法直接搬到 Unity。

## 当前样例输出

已验证输出目录：

```text
D:\Tmp\SubnauticaShaderTest\M_Character_Teeth_ShaderGroups
```

关键文件：

```text
shader_00_idx_556_group_04332.dxil
shader_00_idx_617_group_04318.dxil
shader_00_idx_2524_group_04301.dxil
shader_00_idx_2528_group_04311.dxil
shader_00_idx_3282_group_04160.dxil
shader_00_idx_46521_group_02843.dxil
...
Disasm\shader_00_idx_46521_group_02843.ll
```

Unity 近似版样例：

```text
D:\Tmp\SubnauticaShaderTest\Unity\M_Character_Teeth_Reconstructed.shader
D:\Tmp\SubnauticaShaderTest\Unity\M_Character_Teeth_ReconstructionNotes.md
```

## 推荐后续实现

把当前临时代码整理成正式 exporter：

```powershell
dotnet run --project CUE4Parse\CUE4Parse.Example\CUE4Parse.Example.csproj -c Release -- `
  --game Subnautica2 `
  --paks "D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\Subnautica2\Content\Paks" `
  --mapping "D:\Tmp\Subnautica.2.v.0.10.1.Early.Access\Subnautica2\5.6.1-114707+++Project+SN2-Release-Hotfix-Live-Subnautica2.usmap" `
  --material "/Game/Materials/_Master/Master/M_Character_Teeth" `
  --out "D:\Tmp\ShaderBundles\M_Character_Teeth.bundle"
```

正式 exporter 应实现：

- 输入 material path。
- 自动定位 ResourceHash。
- 自动选择 project shader archive。
- 自动读取 IoStore shader group chunk。
- 自动调用 `decompress_shader.exe` 或内置解压。
- 自动切出 DXIL/DXBC。
- 自动运行 `dxc -dumpbin`。
- 自动生成 manifest/reflection/analysis JSON。
- 可选生成 Unity/GLSL 近似版。

## 复现检查清单

- material JSON 中 `LoadedMaterialResources` 不为空。
- `LoadedShaderMap.ResourceHash` 能在 shader archive `ShaderMapHashes` 中找到。
- target ShaderMap 的 `NumShaders` 大于 0。
- 每个 target shader 的 `ShaderGroupIndex` 能找到对应 `ShaderGroupIoHashes`。
- `IoStoreReader.Read(chunkId)` 能读出 group chunk。
- group 解压后的 `.bin` 大小等于 `ShaderGroupEntries[groupIndex].UncompressedSize`。
- 每个 shader slice 能找到 `DXBC` container magic。
- `dxc -dumpbin` 能成功解析 `.dxil`。
- reflection JSON 中 stage、resources、outputs 非空。

