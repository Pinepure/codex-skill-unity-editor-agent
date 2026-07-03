# Intelligent Capability Iteration

Use this file whenever the manifest lacks a needed Unity operation.

## Goal

Close the immediate task and improve the Unity tool surface without creating unnecessary overlap.

## Closed Loop

### 1. Inventory

- Read current discovery data.
- List the closest matching existing tool IDs.
- Check whether a tool combination already solves the task.

### 2. Gap Statement

Write a short internal gap statement with:

- `goal`: the Unity outcome that must happen
- `existing-tools`: the candidate tool IDs considered
- `insufficient-because`: the precise missing behavior
- `smallest-missing-operation`: one atomic action to add

Only continue if the gap is real.

### 3. Add the Smallest Tool

Design one narrow project-internal tool:

- unique dotted ID
- single responsibility
- explicit args schema
- explicit return schema
- no shell execution unless the user explicitly approves and the tool is marked high risk

Generated tool rules:

- namespace: `AiUnity.EditorAgent.Generated`
- install path: `Assets/Editor/AiUnityEditorAgent/GeneratedTools/`
- installation method: `tool.upsert_script`

### 4. Compile and Refresh

After installing a tool:

1. Poll `compile.status`
2. If compile fails, inspect the best compile diagnostics the current service actually exposes
3. If the first diagnostic is insufficient, inspect `compile.errors`, `compile.errors_since_last_compile`, or service logs as available
4. Fix the generated tool
5. Refresh discovery from the endpoints the current service actually supports, typically `GET /manifest`

Never call the new tool from stale manifest assumptions.

### 5. Execute and Validate

- Call the new tool
- Validate its output with the right inspection or verification tool
- Inspect `service.log_recent` if the tool fails without clear compile errors

### 6. Classify Reuse

At the end of the task, classify the added tool:

- `reuse-now`: the tool is already generally useful and should be reused whenever the manifest advertises it
- `task-specific`: the tool is narrow and only likely useful for this case
- `promote-later`: the gap appears recurring enough that a stable project capability may be worth proposing

Do not merge, refactor, or replace existing project-internal capabilities without user confirmation.

If the result is `task-specific`, prefer removing the generated tool after validation with `tool.delete_generated` so temporary recovery tools do not accumulate in the project.

Common high-value gaps that justify adding a tiny tool instead of stopping:

- binding serialized object references on an existing prefab component
- one-off prefab migration or normalization that must be saved by Unity
- importer-state inspection that current asset tools do not expose
- project-specific runtime or scene inspection needed only for the current feature

## Reuse Rules

- Reuse an existing tool if it can produce the required Unity-side effect or inspection result with acceptable composition.
- Prefer one missing atomic tool over a broad “do everything” tool.
- Prefer a second call to an existing tool over inventing a near-duplicate tool ID.
- If a generated tool already exists in the manifest with equivalent semantics, reuse it.

## Escalate to the User

Ask for confirmation before:

- refactoring or merging existing project-internal capabilities
- broadening an existing tool’s scope to absorb other tools
- changing the behavior or contract of an existing reusable tool
