# Goal: Export UE Cooked Material Bundles From Unity Scene

Objective: export UE cooked material shader bundles for every Unity `.mat` used by a specified Unity scene, then write a batch workspace that can run NoVisual reconstruction without hand-authored material mapping.

This goal only exports cooked data, semantic analysis, shared texture payloads, Agent docs, and batch mapping. Do not reconstruct Unity shaders, restore `.mat` values, run RenderDoc, or do visual validation in this goal.

Write boundary:

```text
Allowed writes:
  the requested batch workspace;
  its child bundles;
  workspace shared Textures payload directory;
  generated Agent docs and batch mapping files inside that workspace.

Not allowed during this scene export goal:
  FModel / CUE4Parse exporter source changes;
  Doc template changes;
  Unity project Assets changes;
  material restore, missing texture import, or .mat writes.
```

## Invocation

Preferred form:

```text
/goal <FModelRepo>\Doc\UE_Cooked_Material_Batch_Export_From_UnityScene_Goal.md UnityScene=<UnityProjectRoot>\Assets\...\Scene.unity
```

Optional:

```text
OutRoot=<Workspace>
Overwrite=true|false
```

If `OutRoot=...` is omitted, use the directory where this goal was started as the batch workspace, unless that directory is the FModel repo or another tool directory. In that case, ask for `OutRoot=...`.

## Matching Rule

The Unity scene path must be inside a Unity `Assets` directory.

The exporter scans the scene and referenced Unity text assets/prefabs for material GUIDs, resolves those GUIDs to Unity `.mat` files, then maps each `.mat` by relative path:

```text
Assets/Art/Environment/Biome/CoralGarden/Rocks/Material/MI_CG_RockPebbles_01a.mat
  ->
/Game/Art/Environment/Biome/CoralGarden/Rocks/Material/MI_CG_RockPebbles_01a
```

Rules:

```text
1. Match by Assets-relative path, not by bare file name.
2. If the exact path-derived /Game material does not exist in cooked UE data, mark that material as not found.
3. Do not substitute a same-name UE material from another folder.
4. Duplicate Unity .mat file names are allowed when their Assets-relative paths differ.
5. When duplicate file names would collide as bundle names, use a path-derived bundle folder name.
```

## Command Shape

Run from the FModel repo:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --game <GameName> `
  --paks "<GamePaks>" `
  --mapping "<GameMapping.usmap>" `
  --unity-scene "<UnityScene.unity>" `
  --out-root "<Workspace>" `
  --include-layer-stack `
  --include-master-modules `
  --include-texture-payload `
  --texture-payload-mode shared `
  --decompress-shader "<decompress_shader.exe>" `
  --overwrite `
  --verbose
```

Use the same game, paks, mapping, and `decompress_shader.exe` values as the normal cooked material bundle export goal.

## Expected Output

The batch workspace should contain:

```text
<Workspace>\*.bundle\
<Workspace>\Textures\
<Workspace>\MaterialMap.json
<Workspace>\MaterialMap.example.json
<Workspace>\summary.json
<Workspace>\batch_manifest.json
<Workspace>\UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md
<Workspace>\commands\verify_all_bundles.ps1
```

`MaterialMap.json` must include each material's Unity `Assets/.../*.mat` path and the path-derived UE `/Game/...` path.

## Verify

After export, verify the batch root:

```powershell
dotnet run --project CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --verify-only "<Workspace>" `
  --verbose
```

Expected:

```text
Verify: OK
```

## Next Step

From the batch workspace:

```text
/goal UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md
```

If the workspace is moved to another machine and `MaterialMap.json` uses `Assets/...` paths, provide:

```text
/goal UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md UnityRoot=<UnityProjectRoot>
```

## Completion Criteria

This goal is complete only when:

```text
1. The scene material GUID scan completed.
2. Every resolved Unity .mat was attempted as a path-derived /Game material.
3. Missing cooked materials are reported explicitly and are not replaced by same-name materials from other folders.
4. Batch workspace docs and MaterialMap.json were generated.
5. Batch root verify succeeded, or failures are reported with actionable material paths.
```
