# UE Shader 还原工作流 - 详细版

这份文档是给人看的操作流程，用来指导从 UE5 cooked 材质数据导出，到 Unity 6 URP Deferred Shader/材质还原、验证和后续修正。

下面统一使用占位符：

```text
<FModelRepo>      FModel 仓库目录
<Workspace>       当前材质逆向工作区
<Bundle>          某个导出的 *.bundle 目录
<MaterialName>    UE 材质名，或者完整 /Game/... 路径
<UnityMatDir>     Unity 工程中某个包含 .mat 的目录
<UnityMat>        Unity 里的目标 .mat 文件
```

## 0. 先确认工作区模式

### 单材质工作区

工作区里只有一个 `.bundle`：

```text
<Workspace>/
  MI_A.bundle/
```

这种方式适合单独还原、单独分发一个材质。

### 多材质工作区

工作区里有多个 `.bundle`：

```text
<Workspace>/
  MI_A.bundle/
  MI_B.bundle/
  Textures/
```

这种方式适合批量导出和批量做第一轮 NoVisual 还原。多个材质时，执行 goal 要带上具体 bundle 路径，避免 AI Agent 处理错材质。

典型流程是：

```text
1. cd <Workspace>
2. 指定 Unity 材质目录，批量导出所有 .mat 对应的 bundle
3. 导出器自动生成 MaterialMap.json
4. 运行 batch NoVisual goal
5. 检查 batch 总结
6. 后续按 bundle 分发给不同的人做视觉验证和效果对齐
```

## 1. 导出材质 bundle

进入材质逆向工作区：

```text
cd <Workspace>
```

普通导出：

```text
/goal <FModelRepo>\Doc\UE_Cooked_Material_Bundle_Export_Goal.md 当前目录导出 <MaterialName> 材质
```

如果需要把贴图 payload 也打进导出结果，使用：

```text
/goal <FModelRepo>\Doc\UE_Cooked_Material_Bundle_With_TexturePayload_Export_Goal.md 当前目录导出 <MaterialName> 材质
```

建议：

```text
单个材质分发给别人：可以使用自包含贴图 payload。
多个材质批量还原：优先使用 workspace 共享 Textures 目录，减少重复贴图。
```

导出完成后，AI Agent 会检查 bundle 是否完整。人只需要确认它最终报告里没有导出失败、验证失败或路径歧义。

## 2. 可选：补 RenderDoc 截帧数据

RenderDoc 不是第一步必须项。建议先完成 cooked bundle 导出和 NoVisual 还原。

如果需要增加运行时证据，把对应材质 drawcall 的导出数据放到该 bundle 下：

```text
<Bundle>\RenderDocCapture\EID_...
```

然后运行：

```text
/goal <Bundle>\UE_RenderDoc_Compact_Summary_Goal.md
```

注意：

```text
多材质工作区里，RenderDocCapture 必须放在对应 bundle 下面。
不要放在 workspace 根目录后让 AI Agent 猜对应哪个材质。
```

## 3. 做 NoVisual 还原

NoVisual 是“不做视觉验证”的第一轮还原。它应该完成：

```text
Shader 结构还原
必要的模块复用或新增
Unity Shader 编译检查
Unity 材质属性恢复
后续视觉验证交接文档
```

它不应该做：

```text
截图对比
RenderDoc 分析
GBuffer capture
主观视觉调参
```

### 单材质

推荐：

```text
/goal <Bundle>\UE_Unity_Material_OneClick_NoVisual_Reconstruction_Goal.md -mat <UnityMat>
```

如果当前 workspace 确实只有一个 bundle，也可以使用 bundle 导出的便捷入口。

### 多材质 batch

多材质批量处理时，从 workspace 根目录开始：

```text
cd <Workspace>
```

指定 Unity 工程里的材质目录，让导出器递归扫描下面所有 `.mat`。

导出器会优先按目录一一对应来找 UE cooked 材质：

```text
Assets/Art/Environment/Biome/CoralGarden/Rocks/Material/MI_CG_RockPebbles_01a.mat
  ->
/Game/Art/Environment/Biome/CoralGarden/Rocks/Material/MI_CG_RockPebbles_01a
```

因此不同目录下有同名 `.mat` 也可以处理；工具会用相对路径判断对应 UE 材质，并在必要时使用带目录信息的 bundle 名避免覆盖。

如果 cooked 数据里只有同名但不同目录的材质，也算没有找到对应材质。批量导出不应该自动改用另一个目录下的同名 UE 材质。

```text
/goal <FModelRepo>\Doc\UE_Cooked_Material_Bundle_Export_Goal.md UnityMatDir=<UnityMatDir>
```

导出后，workspace 里应该类似：

```text
<Workspace>/
  MI_A.bundle/
  MI_B.bundle/
  MI_C.bundle/
  Textures/
  MaterialMap.json
  summary.json
  batch_manifest.json
```

然后直接运行 batch NoVisual：

```text
/goal UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md
```

如果导出的工作区被分发到另一台机器，Unity 工程路径不同，运行时补 `UnityRoot=...`：

```text
/goal UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md UnityRoot=<UnityProjectRoot>
```

如果只想先处理其中一个材质，不跑 batch，也可以单独执行：

```text
/goal MI_A.bundle\UE_Unity_Material_OneClick_NoVisual_Reconstruction_Goal.md -mat <UnityMat for MI_A>
/goal MI_B.bundle\UE_Unity_Material_OneClick_NoVisual_Reconstruction_Goal.md -mat <UnityMat for MI_B>
```

如果因为特殊情况没有 `MaterialMap.json`，才需要手动提供：

```text
/goal UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md MatMap=<YourMaterialMap.json>
```

建议：

```text
可以并行分析多个 bundle。
写 Unity 工程 Assets 时要串行，避免多个 Agent 同时改同一个 Unity 工程。
每个 bundle 完成后，都应该留下交接文档，方便后续其他人继续做视觉验证。
```

batch 完成后，人的检查重点是：

```text
1. 哪些材质成功。
2. 哪些材质失败，需要单独重跑。
3. 哪些材质复用了已有 Shader。
4. 哪些材质新建或扩展了 Shader。
5. 每个 bundle 是否都留下了给后续视觉验证使用的交接总结。
```

后续某个材质要继续做视觉验证时，仍然从 workspace 根目录执行该 bundle 的本地 goal：

```text
/goal MI_A.bundle\UE_Unity_Material_Semantic_Visual_Validation_Goal.md Mat=<UnityMat for MI_A>
```

## 4. 可选：单独恢复 Unity 材质属性

如果 OneClick NoVisual 已经完成材质属性恢复，通常不需要单独跑这一段。

如果只想单独把 UE 材质参数恢复到已有 Unity `.mat`，先跑 DryRun：

```text
/goal <FModelRepo>\Doc\UE_Unity_Material_Property_Restore_Goal.md Mat=<UnityMat> Bundle=<Bundle>
```

确认 AI Agent 报告没有明显问题后，再 Apply：

```text
/goal <FModelRepo>\Doc\UE_Unity_Material_Property_Restore_Goal.md Apply Mat=<UnityMat> Bundle=<Bundle>
```

如果 DryRun 提示缺贴图，不需要人工去看底层 JSON。让 AI Agent 按报告优先使用 bundle/shared texture payload；如果 payload 不存在，再从原始 cooked 数据补导出贴图。等 Unity 导入贴图并生成 `.meta` 后，再重新 DryRun，最后 Apply。

## 5. 视觉验证和效果对齐

视觉验证在材质属性 Apply 之后做。

如果有原游戏参考图，放到：

```text
<Workspace>\VisualRefs\
```

或：

```text
<Workspace>\ScreenShot*
```

然后运行：

```text
/goal <Bundle>\UE_Unity_Material_Semantic_Visual_Validation_Goal.md Mat=<UnityMat>
```

如果没有参考图，但已经手动打开 Unity 场景，并把使用该材质的物体放到 GameView 中央，可以跑轻量 smoke：

```text
/goal <Bundle>\UE_Unity_Material_Lightweight_Visual_Smoke_Goal.md Mat=<UnityMat>
```

视觉验证的目标不是只看“像不像”，还要让 AI Agent 按当前 shader 已实现的功能检查明显错误，例如：

```text
颜色和贴图是否明显错位
法线方向和强度是否明显错误
粗糙度、金属度、AO 是否明显不合理
透明、Mask、Layer Blend、Height Blend 是否明显不对
Vertex Color、Custom Primitive Data、UV 投影等功能是否和场景表现冲突
```

如果只有最终效果图，没有语义通道参考，AI Agent 需要说明哪些部分只是视觉近似，哪些部分已经有证据支持。

## 6. Shader 复用

人不需要手动判断底层 reuse key。只需要知道原则：

```text
如果两个 MI 只是参数不同，应该复用同一个 Unity Shader。
如果 Layer/Blend/Master 逻辑接近，应该复用已有通用模块。
只有逻辑结构确实不同，才应该新建 Shader 或扩展已有 Shader。
```

AI Agent 在还原时会根据 bundle 内分析结果判断：

```text
复用已有 Shader
扩展已有 Shader
新建 Shader
```

如果后续视觉验证发现复用判断错了，应该让 AI Agent 记录原因，并改成扩展或新建。不要为了单个材质粗暴修改已经被其他材质验证过的共享 Shader。

## 7. 本地镜像和版本管理

逆向工作区可以用 Git 做本地版本管理。Unity 工程可能仍然使用 SVN 或团队要求的其他版本管理方式。

建议把本次还原涉及到的 Unity 资源镜像到：

```text
<Workspace>\UnityMirror\Assets\...
```

人只需要关注：

```text
如果 workspace 已经是 Git 仓库，可以用它看 diff 和回退。
不要在 Unity 工程里新建 .git。
不要让多个 Agent 同时改同一个 Unity 工程目录。
```
