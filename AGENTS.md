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
skills/unity6-urp-deferred-shader-reconstruction/SKILL.md
commands/verify_bundle.ps1
commands/refresh_context.ps1
commands/export_again.ps1
```

Required behavior:

- Full export must generate the Agent workspace files automatically.
- `--context-only <existing bundle>` and `--semantic-only <existing bundle>` must repair or refresh the Agent workspace files.
- Batch export must also write a batch-root `AGENTS.md`, `WORKFLOW.md`, `NEXT_TASK.md`, `PROMPT_NEXT_SESSION.md`, `agent_context.json`, local skill, and verification script.
- `--verify-only` must fail if the required single-bundle Agent workspace files are missing or if `agent_context.json` is not parseable.
- The generated target must remain Unity 6 URP Deferred with DOTS instancing unless the user explicitly changes the reconstruction target.
- Generated Agent docs must tell future Agents to read `analysis/ai_context_pack.md/json` and `analysis/reconstruction_entrypoints.json` before raw `shaders/*.dxil.ll`.
- Generated Agent docs must state that UE Deferred LightPass logic should be reused through Unity URP Deferred lighting unless `analysis/lightpass_audit.json` proves game-specific custom lighting.

Do not treat the root Agent docs as optional packaging metadata. They are part of the shader reconstruction handoff contract.
