---
name: unity-editor-agent
description: Control a Unity project through the local AI Unity Editor Agent service with a strict Unity-tool-first workflow. Use when Codex needs to inspect, query, modify, or extend Unity assets, prefabs, scenes, compile state, or project-internal editor tooling through `/health`, manifest discovery endpoints, `/tool/describe_many`, `/result/{handleId}`, and `/call/{toolId}`. Prefer this skill whenever a Unity task can be executed through manifest-advertised tools instead of shell edits or direct asset file edits. Add a new Unity capability only after confirming the latest discovery results do not already provide a sufficient tool or tool combination, never modify package code, and require user confirmation before refactoring, merging, or changing existing project-internal capabilities.
---

# Unity Editor Agent

## Overview

Use the AI Unity Editor Agent as the primary execution surface for Unity work. Treat `manifestHash` as the capability cache key, prefer search, bundle, describe, and paged result flows over repeatedly loading the full manifest, and add only the smallest missing project-internal tool when discovery proves a real gap.

## Core Loop

1. Start every Unity task with `GET /health` and cache `manifestHash`.
2. Reuse cached capability knowledge while `manifestHash` stays unchanged.
3. Prefer `POST /manifest/search` or `GET /manifest/bundle/{id}` to narrow candidate tools.
4. Use `POST /tool/describe_many` before calling unfamiliar tools.
5. Use existing Unity tools first.
6. Add a new tool only when discovery lacks a sufficient existing tool or tool combination.
7. After `tool.upsert_script`, wait for compile idle, inspect compile summaries if needed, re-check `manifestHash`, refresh discovery, then call the new tool.
8. Validate the result with the relevant playbook and the validation matrix before closing the task.

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
2. Re-check `manifestHash`, then reload discovery with search, bundle, or full manifest only if needed.
3. Call the new tool.
4. Validate output.
5. Decide whether the tool should be reused as-is, treated as one-off, or proposed later as a stable project capability.

Read [references/capability-iteration.md](references/capability-iteration.md) for the detailed gap-analysis and reuse loop.

## Task Routing

- For prefab inspection, prefab wiring, nested prefabs, or prefab edits: read [references/prefab-playbook.md](references/prefab-playbook.md)
- For asset search, GUID/path work, dependencies, reverse references, and text asset reads: read [references/asset-playbook.md](references/asset-playbook.md)
- For scene object creation or scene usage inspection: read [references/scene-playbook.md](references/scene-playbook.md)
- For compile problems, import-state confusion, or service/tool failures: read [references/diagnostics-playbook.md](references/diagnostics-playbook.md)

## High-Frequency Pitfalls

- When moving UI art out of generated folders such as `VectoUI`, inspect the consuming prefab dependencies first, then move the assets, update importer settings, and create or refresh any needed `SpriteAtlas`.
- For full-screen panel, mask, or dimmer sprites, prefer `Sprite` import mode with `spriteMeshType = FullRect` instead of tight mesh, then verify the importer state with a Unity-side asset tool.
- Do not try to reparent children of a nested prefab instance inside another prefab. First inspect the nesting with `prefab.nested_prefab_overrides`. If hierarchy changes are required, normalize the source prefab and reinstall the nested instance, or unpack only with explicit user approval.
- Treat task-specific generated tools as disposable. After the task succeeds, either delete them with `tool.delete_generated` or explicitly classify them as reusable project capabilities.

## Validation

Use [references/validation-matrix.md](references/validation-matrix.md) to decide what to verify before finishing.

At minimum:

- Re-check compile state after adding or changing tools.
- Re-check `manifestHash` and refresh discovery after any tool installation or tool change.
- For write operations, verify the resulting Unity state with a manifest-advertised inspection tool whenever possible.
- When a tool fails, inspect `compile.snapshot`, `compile.errors_summary`, or `service.log_recent` before guessing.
