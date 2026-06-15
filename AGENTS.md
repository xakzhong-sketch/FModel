# Project Agent Notes

## ShaderBundleExporter Workspace Contract

When working on `CUE4Parse/CUE4Parse.ShaderBundleExporter`, every exported cooked shader bundle must be usable by a fresh AI Agent without relying on external chat history.

Each single-bundle export directory must include these root files:

```text
README.md
AGENTS.md
WORKFLOW.md
NEXT_TASK.md
PROMPT_NEXT_SESSION.md
agent_context.json
UE_RenderDoc_Compact_Summary_Goal.md
UE_Unity_Material_Semantic_Visual_Validation_Goal.md
UE_Unity_Material_Lightweight_Visual_Smoke_Goal.md
UE_Unity_Material_OneClick_Reconstruction_Goal.md
UE_Unity_Material_OneClick_NoVisual_Reconstruction_Goal.md
skills/unity6-urp-deferred-shader-reconstruction/SKILL.md
commands/verify_bundle.ps1
commands/refresh_context.ps1
commands/export_again.ps1
```

Required behavior:

- Full export must generate the Agent workspace files automatically.
- `--context-only <existing bundle>` and `--semantic-only <existing bundle>` must repair or refresh the Agent workspace files.
- For a single-bundle workspace whose parent directory contains exactly one `.bundle`, full export and `--context-only` must also generate workspace-root goal launchers so future Agents can run `/goal <GoalFile>.md` without writing `.bundle\...` relative paths. If zero or multiple bundles exist, generated docs must require `Bundle=...` instead of guessing.
- Batch export must also write a batch-root `AGENTS.md`, `WORKFLOW.md`, `NEXT_TASK.md`, `PROMPT_NEXT_SESSION.md`, `UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md`, `agent_context.json`, `batch_manifest.json`, `MaterialMap.example.json`, local skill, and verification script.
- `--verify-only` must fail if the required single-bundle Agent workspace files are missing or if `agent_context.json` is not parseable.
- `--verify-only` must also support batch workspace roots by validating `batch_manifest.json`, `summary.json`, batch Agent docs, and parseable JSON entrypoints.
- Multi-bundle workspaces must not expose ambiguous root-level bundle-local goal launchers. Use explicit `Bundle\Goal.md` paths or `UE_Workspace_Batch_NoVisual_Reconstruction_Goal.md`.
- New RenderDoc drawcall exports belong under the target bundle at `Bundle/RenderDocCapture`; workspace-root RenderDocCapture is legacy fallback only.
- Texture payload mode must support `shared`, `bundle`, and `none`. Shared mode writes workspace `Textures/manifest.json` plus per-bundle `texture_payload_manifest.json`; bundle mode writes self-contained `texture_payload/manifest.json`.
- The generated target must remain Unity 6 URP Deferred with DOTS instancing unless the user explicitly changes the reconstruction target.
- Generated Agent docs must tell future Agents to read `analysis/ai_context_pack.md/json` and `analysis/reconstruction_entrypoints.json` before raw `shaders/*.dxil.ll`.
- Generated Agent docs must state that UE Deferred LightPass logic should be reused through Unity URP Deferred lighting unless `analysis/lightpass_audit.json` proves game-specific custom lighting.

Do not treat the root Agent docs as optional packaging metadata. They are part of the shader reconstruction handoff contract.
