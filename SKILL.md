---
name: unity-editor-agent
description: Control a Unity project through the local AI Unity Editor Agent service with a strict Unity-tool-first workflow. Use when Codex needs to inspect, query, modify, or extend Unity assets, prefabs, scenes, compile state, or project-internal editor tooling through `/health`, `GET /manifest`, `/call/{toolId}`, optional result handles, and project-generated Editor tools. Prefer this skill whenever a Unity task can be executed through manifest-advertised tools instead of shell edits or direct asset file edits. Add a new Unity capability only after confirming the latest discovery results do not already provide a sufficient tool or tool combination, never modify package code, and require user confirmation before refactoring, merging, or changing existing project-internal capabilities.
---

# Unity Editor Agent

## Overview

Use the AI Unity Editor Agent as the primary execution surface for Unity work. Start with `/health`, discover tools from `GET /manifest`, and add only the smallest missing project-internal tool when discovery proves a real gap. Some projects still run agent versions such as `0.1.1` that do not expose `manifestHash`, `/manifest/search`, bundle endpoints, or richer compile-summary endpoints, so the workflow must degrade cleanly to the tool list that actually exists today.

## Core Loop

1. Start every Unity task by reading `endpoint.json` and `token.txt`, then call `GET /health`.
2. Read `GET /manifest` and treat that returned tool list as the source of truth for the current session.
3. If the running service exposes richer discovery endpoints such as `/manifest/search`, bundle endpoints, or `/tool/describe_many`, use them. If not, continue from `GET /manifest` without assuming those routes exist.
4. Use existing Unity tools first.
5. If the manifest proves the capability is missing, add the smallest possible generated tool for that one Unity-side operation.
6. After `tool.upsert_script`, wait for compile idle, reload `GET /manifest`, confirm the new tool ID is present, then call it directly or through `batch.run`.
7. For multi-step tasks, prefer worksets, snapshots, and `batch.run` instead of many small round trips.
8. Before planning from scratch, query `workflow.match_recipes` for similar successful flows whenever the task is not trivial.
9. Validate with `Unity + Git + AI`: capture a baseline, execute the task, run validators, then interpret the result with Git scope and the task goal before closing.
10. After a validated success on a reusable task, record it with `workflow.record_success` so future runs can reuse the same path.

## Compatibility Notes

- Do not assume a fixed Unity port. Read `<ProjectRoot>/Library/AiUnityEditorAgent/endpoint.json` for the current endpoint.
- The current local service may require `X-Unity-Ai-Token`; do not assume Bearer auth.
- `manifestHash` is useful when available, but some versions do not return it. In those versions, re-read `/manifest` after tool installation or other capability changes instead of pretending a cache key exists.
- If `/manifest/search`, bundle routes, or `/tool/describe_many` are unavailable, that is not a blocker. Fall back to `GET /manifest` and the advertised tool IDs.
- If `compile.snapshot` or `compile.errors_summary` do not exist, use `compile.status`, `compile.errors`, `compile.errors_since_last_compile`, `service.log_recent`, and `service.call_recent`.
- If worksets, batch execution, or validator tools exist, prefer them over repeatedly pulling large lists or manually deciding that a task is done.
- If no existing tool can modify the required Unity-managed state, you are expected to add a small project-internal Editor tool rather than stopping at analysis.
- For one-off prefab binding or migration work, prefer a disposable generated tool. Execute it, validate the serialized result from Unity, then delete it.

Read [references/ai-unity-editor-agent-protocol.md](references/ai-unity-editor-agent-protocol.md) when exact HTTP details, generated tool contract rules, or default endpoints are needed.

## Non-Negotiables

- Never assume a tool exists before reading current discovery data.
- Never bypass a manifest-advertised Unity tool with shell edits, direct `.prefab` / `.unity` file edits, or other side channels when that Unity tool already covers the operation.
- Only add capabilities when existing manifest tools are insufficient for the specific task.
- Keep capability growth additive by default. Do not refactor, merge, rename, broaden, or otherwise modify an existing project-internal Unity capability unless the user explicitly confirms.
- Only change project-internal code. Never modify `Packages/`.
- Generated tools must be installed through `tool.upsert_script` and live under `Assets/Editor/AiUnityEditorAgent/GeneratedTools/`.
- Respect `requiresConfirmation`. Never bypass it.
- When a tool returns `resultHandle`, continue through `/result/{handleId}` instead of re-running the tool with larger payload limits.
- Prefer small, single-purpose tools over broad unrestricted ones.
- Do not hand-edit `.prefab`, `.unity`, or other Unity-serialized YAML when the goal is to change Unity-managed references or hierarchy. Add or reuse an Editor tool and let Unity serialize the change.

Read [references/decision-policy.md](references/decision-policy.md) when the right path is ambiguous.
Read [references/pitfalls-log.md](references/pitfalls-log.md) when the task involves prefab hierarchy surgery, generated UI resources, importer-sensitive asset moves, or temporary project-specific editor helpers.

## Intelligent Capability Iteration

The skill is not only for executing Unity tasks. It must also improve the tool surface safely.

Before adding a tool:

1. List the candidate manifest tool IDs that are closest to the task.
2. State why each candidate is insufficient.
3. Define the smallest missing operation.
4. Choose an additive tool ID that does not overlap an existing tool unnecessarily.

After adding a tool:

1. Wait for compile idle.
2. Reload discovery from the currently supported endpoints. If `manifestHash` exists, use it as a cache key; otherwise simply re-read `GET /manifest`.
3. Call the new tool directly or through `batch.run` if the task is multi-step.
4. Validate output through `validator.capture_baseline` and `validator.run` when write operations are involved.
5. If the task succeeded and the flow is reusable, call `workflow.record_success` with the goal, step tool ids, validator ids, and changed files.
6. Decide whether the tool should be reused as-is, treated as one-off, or promoted into a stable workflow recipe.

Read [references/capability-iteration.md](references/capability-iteration.md) for the detailed gap-analysis and reuse loop.

## Task Routing

- For prefab inspection, prefab wiring, nested prefabs, or prefab edits: read [references/prefab-playbook.md](references/prefab-playbook.md)
- For asset search, GUID/path work, dependencies, reverse references, and text asset reads: read [references/asset-playbook.md](references/asset-playbook.md)
- For scene object creation or scene usage inspection: read [references/scene-playbook.md](references/scene-playbook.md)
- For compile problems, import-state confusion, or service/tool failures: read [references/diagnostics-playbook.md](references/diagnostics-playbook.md)

## High-Frequency Pitfalls

- When moving UI art out of generated folders such as `VectoUI`, inspect the consuming prefab dependencies first, then move the assets, update importer settings, and create or refresh any needed `SpriteAtlas`.
- For full-screen panel, mask, or dimmer sprites, prefer `Sprite` import mode with `spriteMeshType = FullRect` instead of tight mesh, then verify the importer state with a Unity-side asset tool.
- When generating raster sprite art through the built-in image generation path and alpha is needed, prefer a perfectly flat chroma-key green background, remove that background locally, then import the resulting PNG as a Unity sprite. For frame animation, assemble the final `n x m` sheet after chroma-key removal so each cell keeps clean alpha edges.
- For URP particle materials that consume alpha textures such as flames, embers, or fireflies, explicitly set the material `SurfaceType` to `Transparent`. A correct PNG alpha channel alone is not enough; leaving the material opaque can produce black fringes or dark quads around the particle.
- Do not try to reparent children of a nested prefab instance inside another prefab. First inspect the nesting with `prefab.nested_prefab_overrides`. If hierarchy changes are required, normalize the source prefab and reinstall the nested instance, or unpack only with explicit user approval.
- Treat task-specific generated tools as disposable. After the task succeeds, either delete them with `tool.delete_generated` or explicitly classify them as reusable project capabilities.
- For serialized prefab object references such as sprites or child VFX prefabs, bind them through a generated Editor tool that loads prefab contents, assigns component fields, and saves through `PrefabUtility`. Do not rely on direct YAML edits.
- After prefab asset wiring, validate from the consumer side with tools such as `asset.dependencies` and `prefab.find_asset_references`, not just by reading the script source.

## Validation

Use [references/validation-matrix.md](references/validation-matrix.md) to decide what to verify before finishing.

At minimum:

- Re-check compile state after adding or changing tools.
- Re-check the live discovery surface after any tool installation or tool change. Use `manifestHash` when the service exposes it; otherwise re-read `GET /manifest`.
- For write operations, verify the resulting Unity state with a manifest-advertised inspection tool whenever possible.
- For write operations, prefer the validation loop: baseline -> execute -> validator.run -> inspect Git scope -> decide whether the task is complete.
- Read `validator.run` scope fields such as `checkedFiles`, `unstagedFiles`, `stagedFiles`, and `untrackedFiles` before deciding the task is complete.
- When the task produced a reusable successful path, finish by calling `workflow.record_success`.
- When a tool fails, inspect the best diagnostics the current service actually exposes, such as `compile.status`, `compile.errors`, `compile.errors_since_last_compile`, `service.log_recent`, or `service.call_recent`.
