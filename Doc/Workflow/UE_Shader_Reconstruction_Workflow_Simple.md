# UE Shader 还原工作流 - 极简版

这份是给人看的简化流程。

占位符：

```text
<FModelRepo>    FModel 仓库目录
<Workspace>     当前材质逆向工作区
<Bundle>        导出的 .bundle 目录
<MaterialName>  UE 材质名，或完整 /Game/... 路径
<UnityMatDir>   Unity 工程中某个包含 .mat 的目录
<UnityScene>    Unity 工程中的 .unity 场景文件
<UnityMat>      Unity 里的目标 .mat
```

## 单个材质

进入工作区：

```text
cd <Workspace>
```

导出 bundle：

```text
/goal <FModelRepo>\Doc\UE_Cooked_Material_Bundle_Export_Goal.md 当前目录导出 <MaterialName> 材质
```

如果希望 bundle 自带贴图 payload：

```text
/goal <FModelRepo>\Doc\UE_Cooked_Material_Bundle_With_TexturePayload_Export_Goal.md 当前目录导出 <MaterialName> 材质
```

可选：补 RenderDoc 证据：

```text
把 drawcall 导出数据放到 <Bundle>\RenderDocCapture\EID_...
/goal <Bundle>\UE_RenderDoc_Compact_Summary_Goal.md
```

不做视觉验证的第一轮还原：

```text
/goal <Bundle>\UE_Unity_Material_OneClick_NoVisual_Reconstruction_Goal.md -mat <UnityMat>
```

如果需要单独恢复 Unity 材质属性：

```text
/goal <FModelRepo>\Doc\UE_Unity_Material_Property_Restore_Goal.md Mat=<UnityMat> Bundle=<Bundle>
/goal <FModelRepo>\Doc\UE_Unity_Material_Property_Restore_Goal.md Apply Mat=<UnityMat> Bundle=<Bundle>
```

如果需要视觉验证：

```text
把参考图放到 VisualRefs 或 ScreenShot*
/goal <Bundle>\UE_Unity_Material_Semantic_Visual_Validation_Goal.md Mat=<UnityMat>
```

没有参考图，但 Unity 场景里目标物体已经在 GameView 中央：

```text
/goal <Bundle>\UE_Unity_Material_Lightweight_Visual_Smoke_Goal.md Mat=<UnityMat>
```

## 多个材质

进入工作区：

```text
cd <Workspace>
```

第一步：指定 Unity 材质目录，批量导出 bundle：

```text
/goal <FModelRepo>\Doc\UE_Cooked_Material_Bundle_Export_Goal.md UnityMatDir=<UnityMatDir>
```

或者只导出某个场景用到的材质：

```text
/goal <FModelRepo>\Doc\UE_Cooked_Material_Batch_Export_From_UnityScene_Goal.md UnityScene=<UnityScene>
```

工具会按相对路径映射：

```text
Assets/.../MI_Name.mat -> /Game/.../MI_Name
```

所以不同目录下同名 `.mat` 也可以处理。

如果 UE cooked 数据里只有同名但不同目录的材质，算找不到，不做同名兜底。

导出后工作区结构类似：

```text
<Workspace>/
  MI_A.bundle/
  MI_B.bundle/
  MI_C.bundle/
  Textures/
  MaterialMap.json
```

第二步：批量做 NoVisual：

```text
/goal UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md
```

这一步默认会生成 shader、做材质 DryRun、自动从 bundle/shared texture payload 导入缺失贴图、跑 Unity 编译检查并输出交接文档；不做视觉验证，也不会 Apply 写 `.mat`。Unity 版本从目标工程的 `ProjectSettings/ProjectVersion.txt` 读取。

batch 报告里的 `StaticReconstructionCoverageStatus` 如果是 `needs_static_reconstruction`，表示当前只是可编译 scaffold，导出数据里已有的模块/功能还没完整覆盖，不能当成还原完成。

缺失贴图默认导入到：

```text
Assets/Art/Recovered/Subnautica2
```

导入后会刷新 Unity 生成 `.meta`，再重新 DryRun。

如果要真正替换 Unity 工程里的 `.mat` shader，再跑：

```text
/goal UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md Apply
```

没有编译成功证据、还有 `MissingTextureGuids`、或贴图还没导入 Unity 生成 `.meta` 的 bundle 不应该 Apply。

如果手动 Apply 后贴图全是 `None`，先还原 `.mat` 的 `.bak` / 版本库，再补贴图并重新 Apply。

换机器后 Unity 工程路径不同，就加：

```text
/goal UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md UnityRoot=<UnityProjectRoot>
```

如果不想批量，也可以逐个做：

```text
/goal MI_A.bundle\UE_Unity_Material_OneClick_NoVisual_Reconstruction_Goal.md -mat <UnityMat for MI_A>
/goal MI_B.bundle\UE_Unity_Material_OneClick_NoVisual_Reconstruction_Goal.md -mat <UnityMat for MI_B>
```

第四步：后续单个材质视觉验证：

```text
把参考图放到 VisualRefs 或 ScreenShot*
/goal MI_A.bundle\UE_Unity_Material_Semantic_Visual_Validation_Goal.md Mat=<UnityMat for MI_A>
```

RenderDoc 数据也放到对应 bundle 下，再跑对应 bundle 的 summary：

```text
MI_A.bundle\RenderDocCapture\EID_...
/goal MI_A.bundle\UE_RenderDoc_Compact_Summary_Goal.md
```

## 人只需要注意

```text
缺贴图：让 AI Agent 按报告优先使用 bundle/shared texture payload，必要时写入指定的 Unity 贴图输出目录。
Shader 复用：参数不同不应该新建 Shader；逻辑确实不同才扩展或新建。
NoVisual：不做视觉调参，但不能只是能编译；要覆盖导出数据里已有的 Layer / Blend / Function / static feature 证据。覆盖不足时应标成 `needs_static_reconstruction`。
视觉验证：有参考图就跑语义验证；没有参考图但场景已摆好就跑轻量 smoke。
多材质：不要让多个 Agent 同时写同一个 Unity 工程 Assets 目录。
工具源码：导出/还原阶段不要修改 FModel、CUE4Parse 或 Doc 模板，除非当前目标明确是修工具。
```
