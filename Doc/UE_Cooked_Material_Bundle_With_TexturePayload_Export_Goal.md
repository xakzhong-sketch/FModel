# Goal: Export UE Cooked Material Bundle With Texture Payload

Objective: export one UE cooked material bundle and include an optional decoded texture payload so the bundle can be distributed to Agents or users who do not have the original UE cooked game data.

Use this Goal as the first workflow step when the bundle should be self-contained for missing Unity texture recovery.

For multi-bundle workspaces, prefer the shared workspace texture store instead:

```text
--include-texture-payload --texture-payload-mode shared
```

Shared mode writes payloads to workspace `Textures/` and per-bundle references to `texture_payload_manifest.json`. This self-contained Goal uses `--texture-payload-mode bundle`.

## When To Use

Choose between these first-step export Goals:

```text
Lightweight bundle:
  /goal <FModelRepo>/Doc/UE_Cooked_Material_Bundle_Export_Goal.md current directory export <MaterialName> material

Bundle with decoded texture payload:
  /goal <FModelRepo>/Doc/UE_Cooked_Material_Bundle_With_TexturePayload_Export_Goal.md current directory export <MaterialName> material
```

Use this payload Goal when:

```text
1. The exporting machine has access to original UE cooked game data.
2. The resulting bundle may be distributed to users or Agents without original game paks/containers.
3. Later Unity material restore should be able to recover missing texture assets from the bundle itself.
```

Do not use this Goal just for shader logic analysis when bundle size matters and downstream users can still access original cooked game data.

## Output Contract

The output bundle must contain the normal cooked shader bundle plus:

```text
<Bundle>/texture_payload/manifest.json
<Bundle>/texture_payload/Game/.../T_Name.png
<Bundle>/texture_payload/Game/.../T_Name.png.ue_texture_export.json
```

Important distinction:

```text
source/textures/*.cooked.json
  cooked texture metadata only; not Unity-importable image files.

texture_payload/*.png
  decoded Unity-importable texture payload files.
```

## Required Steps

1. Resolve the requested material to a unique `/Game/...` path using the normal bundle export Goal rules.
2. Run the normal cooked material bundle export.
3. Verify the bundle.
4. Run texture payload export with `--payload-scope referenced_textures`.
5. Verify the bundle again.
6. Report the final bundle path, payload manifest path, exported/skipped/failed payload counts, and verify result.

## Command Shape

Normal bundle export uses the existing exporter command shape from:

```text
<FModelRepo>/Doc/UE_Cooked_Material_Bundle_Export_Goal.md
```

After that succeeds, run:

```powershell
dotnet run --project <FModelRepo>\CUE4Parse\CUE4Parse.ShaderBundleExporter\CUE4Parse.ShaderBundleExporter.csproj -c Release -- `
  --export-bundle-texture-payload `
  --game "<GameName>" `
  --paks "<GamePaks>" `
  --mapping "<GameMapping.usmap>" `
  --bundle "<Bundle>" `
  --texture-payload-mode bundle `
  --payload-scope referenced_textures
```

Optional:

```text
--overwrite-textures
  Regenerate existing payload PNG files.

--payload-out texture_payload
  Optional bundle-local payload output folder. Default is texture_payload.
```

For batch or one-command export implementations, `--include-texture-payload --payload-scope referenced_textures` may be used when supported. The final output must be equivalent to running normal export followed by `--export-bundle-texture-payload`.

For a multi-bundle workspace, use:

```text
--include-texture-payload --texture-payload-mode shared
```

Expected shared output:

```text
<Workspace>/Textures/manifest.json
<Workspace>/Textures/payload/Game/.../T_Name.png
<Bundle>/texture_payload_manifest.json
```

## Safety Rules

```text
1. Payload export requires original UE cooked game data.
2. Payload consumption does not require original UE cooked game data.
3. Do not run texture payload export during PROMPT_NEXT_SESSION.md shader reconstruction.
4. Do not modify Unity .mat files in this Goal.
5. Do not generate Unity .meta files in this Goal.
6. Do not export unrelated game textures; use referenced_textures from the current bundle.
7. Do not write machine-local absolute paths into bundle Markdown, agent_context.json, or texture_payload/manifest.json.
8. Existing payload texture files are skipped unless --overwrite-textures is explicit.
```

## After Export

Shader reconstruction still starts from:

```text
/goal <Bundle>/PROMPT_NEXT_SESSION.md
```

Material restore still runs separately:

```text
/goal <FModelRepo>/Doc/UE_Unity_Material_Property_Restore_Goal.md Mat=<Unity .mat> Bundle=<Bundle>
```

If material restore DryRun reports missing texture GUIDs and `texture_payload/manifest.json` or `texture_payload_manifest.json` exists, import missing textures from payload before asking for original game data.

## Completion Criteria

This Goal is complete only when:

```text
1. The normal bundle export completed.
2. <Bundle>/texture_payload/manifest.json exists and is parseable.
3. Payload manifest has Schema = ue-bundle-texture-payload/v1.
4. Payload manifest paths are bundle-relative and portable.
5. Each exported/skipped payload item has an existing PNG and sidecar.
6. --verify-only <Bundle> returns Verify: OK.
7. Final response reports exported/skipped/failed/missing-object-path counts.
8. If payload export is partial, final response identifies failed or missing texture ObjectPaths.
```
