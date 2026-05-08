# Decision Policy

Use this file when the execution path is ambiguous or when the task might tempt you to bypass the Unity tool loop.

## Mandatory Ordering

For every Unity task:

1. `GET /health`
2. Cache `manifestHash` and reuse capability knowledge while it is unchanged.
3. `POST /manifest/search` or `GET /manifest/bundle/{id}`
4. `POST /tool/describe_many` for the tools you may actually call
5. `compile.status`
6. Choose the best existing tool or tool combination from current discovery results.
7. Only if discovery is insufficient, add one new project-internal tool.
8. Wait for compile idle, re-check `manifestHash`, and refresh discovery.
9. Call the new or existing tool.
10. Validate.

## Unity Tool First

Use Unity tools for Unity-managed state whenever possible:

- Prefabs
- Scenes
- AssetDatabase-backed lookups
- Compile diagnostics
- Tool installation and capability extension

Use non-Unity tools only for:

- Reading local repository code to understand project architecture
- Authoring the C# source that will be uploaded through `tool.upsert_script`
- Reading the protocol or skill reference files
- Inspecting artifacts for which no Unity tool exists yet

Do not use non-Unity tools to modify Unity-managed state if a manifest tool already supports the operation.

## Additive-Only Capability Policy

Default rule: add new capability, do not modify existing capability.

Before changing an existing project-internal capability, stop unless one of these is true:

- The user explicitly asked to change that existing capability
- The change is a direct fix inside a newly added generated tool for the current task

If the work would refactor, merge, rename, broaden, or otherwise reshape an existing project-internal capability, ask the user first.

Never modify `Packages/`.

## Capability Gap Proof

Before adding a tool, explicitly prove the gap:

1. State the concrete Unity outcome required.
2. List the candidate tool IDs already present in the manifest.
3. State why each candidate or combination is insufficient.
4. Define the smallest missing operation.
5. Choose a unique, stable, additive tool ID.

If an existing generated tool already covers the gap, reuse it instead of creating a near-duplicate.

## Schema Discipline

Describe-many schema fields are JSON-encoded strings. Parse them before deciding:

- which parameters a tool accepts
- whether a tool can cover the task
- whether a new tool would overlap an existing one

Do not infer arguments from tool names alone.
