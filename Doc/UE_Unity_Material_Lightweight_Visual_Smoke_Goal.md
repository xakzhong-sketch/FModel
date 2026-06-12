# Goal: Unity Material Lightweight Visual Smoke Validation

Use this goal after Unity shader reconstruction and after the material restore goal has been run with explicit `Apply`.

This is a no-reference current-scene validation pass. It does not prove UE visual parity. It catches obvious rendering failures and obvious contradictions between the reconstructed shader's implemented features and what is visible in Unity.

## Invocation

Run this from the material workspace, bundle parent, or bundle directory:

```text
/goal <FModelRepo>\Doc\UE_Unity_Material_Lightweight_Visual_Smoke_Goal.md Mat=<absolute or Assets-relative .mat> Bundle=<bundle>
```

If the bundle provides a local copy, prefer:

```text
/goal <Bundle>\UE_Unity_Material_Lightweight_Visual_Smoke_Goal.md Mat=<absolute or Assets-relative .mat> Bundle=<Bundle>
```

## Preconditions

The user should manually open the Unity scene that contains an object using the target material and place that object near the center of GameView.

Use the current Unity editor process when possible. Do not launch a second Unity process if the project is already open.

## Scope

This goal may read:

- the target bundle and compact analysis files;
- the target `.mat`, assigned shader, directly referenced shader/module files, texture import metadata, and latest restore reports;
- current-scene GameView screenshot and optional decoded semantic captures produced for this validation run.

This goal must not:

- export a new bundle;
- run RenderDoc compact summary;
- restore material properties or export missing textures;
- recursively inspect the full Unity `Assets` tree;
- claim high-fidelity UE parity without semantic references or RenderDoc/cooked proof.

## Required Read Order

Read compact evidence first:

1. `AGENTS.md`
2. `analysis/ai_context_pack.md`
3. `analysis/unity_layer_reconstruction_contract.json`
4. `analysis/material_layer_stack.json`
5. `analysis/material_layer_parameter_bindings.json`
6. `analysis/material_static_permutation.json`
7. `analysis/unity_shader_assignment.json`
8. `parameters/material_parameters.json`
9. `parameters/textures.json`
10. latest material restore report, if present
11. latest shader reconstruction report or assumptions report, if present

Open selected shader/module files only after the compact contract is understood.

## Validation Gate

First confirm the non-visual gate:

- Unity imports/compiles the assigned shader with no shader errors.
- The target material is not using Unity's error shader.
- The material restore report has no blocking errors.
- Critical texture GUIDs required by visible implemented features are not missing.

If this gate fails, stop and report `Fail`. Do not proceed to screenshot interpretation.

## Current-Scene Capture

Capture the currently open scene after the user has centered the target object in GameView.

Preferred outputs:

```text
UnityValidation/lightweight_smoke/
  gameview.png
  albedo.png                  optional decoded semantic output
  normal_world.png            optional decoded semantic output
  metallic.png                optional decoded semantic output
  smoothness.png              optional decoded semantic output
  occlusion.png               optional decoded semantic output
  report.md
  report.json
```

If decoded semantic captures are unavailable, use `gameview.png` only and mark channel-specific conclusions as `Inconclusive` unless the failure is visually obvious.

## Smoke Checks

Treat the bundle contract and shader code as the expected implemented feature set. A check can fail when the shader claims to implement a feature and the Unity render clearly contradicts it.

Required checks:

- Target coverage: the target object must occupy a meaningful region near the center. If not, ask the user to recenter the scene and rerun.
- Error color: pink/magenta error shader, missing include fallback, NaN garbage, extreme overexposure, or fully black/white output without material evidence is `Fail`.
- Visibility: a masked/opaque material that should be visible but is fully clipped/transparent is `Fail`.
- Base color / BCM: if the shader implements base-color texture sampling and the material has a bound base-color texture, a flat fallback color is `Fail` unless the restore report proves that texture is intentionally missing or disabled.
- Normal / NRH: if the shader implements normal/height/NRH sampling and semantic `normal_world.png` is available, a flat or obviously invalid normal field is `Fail`. From final lit only, classify suspicious flatness as `Needs Evidence`, not a hard formula proof.
- Layer/blend stack: if multiple active layers/blends/textures are present in `material_layer_stack.json` and the shader claims to implement them, a single constant output with no visible layer contribution is `Fail` when key textures are bound.
- Tint/curve/gradient/custom primitive data: if the shader implements these branches and restored parameters are non-default or runtime evidence says they are active, their visible effect must not be absent without explanation. If the current scene lacks the required CPD or mesh data, mark `Needs Runtime Evidence` rather than silently passing.
- Alpha/mask/cutout: if opacity mask or alpha clip is implemented, check for plausible coverage instead of all-on/all-off failure.
- UV/projection/triplanar: obvious texture swimming, zero UVs, or one-pixel tiling across the object is `Fail` when the shader claims to implement a stable UV/projection path.
- Texture binding: if the material uses missing placeholder textures for core visible slots, `Fail` even if the shader compiles.

## Classification

Use exactly one final classification:

- `Pass`: no compile/import errors, target is visible, no obvious contradiction between implemented shader features and current render.
- `Fail`: at least one required check fails with concrete evidence.
- `Needs Evidence`: capture setup, missing runtime data, missing texture imports, or lack of semantic channels prevents a reliable smoke conclusion.

## Fix Discipline

This goal can identify issues. It may modify shader/material files only if the user's current instruction explicitly asks for correction, not just validation.

Before any fix, prove:

1. Which semantic feature is wrong: BaseColor, Normal, Roughness/Smoothness, Metallic/AO, Alpha/Mask, LayerBlend, HeightBlend, VertexColor, CPD/per-instance data, UV/projection/triplanar, or renderer-only behavior.
2. The shader/material actually claims to implement that feature.
3. The current Unity render contradicts that implemented feature.
4. The smallest fix and the expected validation signal after the fix.

Do not assume Unity mesh color channels match UE post-VS color channels. Compare available UE/runtime evidence and Unity mesh data before applying vertex-color swizzles or mesh-driven fixes.

## Completion Criteria

Write `UnityValidation/lightweight_smoke/report.md` and `report.json` when possible.

Final response must include:

- bundle path and material path;
- Unity shader compile/import status;
- capture output directory;
- final classification: `Pass`, `Fail`, or `Needs Evidence`;
- failed/suspicious checks grouped by semantic feature;
- whether any issue is a shader logic problem, material/texture binding problem, scene/capture setup problem, or missing runtime evidence.
