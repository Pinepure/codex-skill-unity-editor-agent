# Prefab Playbook

Use this file for prefab inspection, prefab creation, nested prefab analysis, UI wiring, and prefab-side capability gaps.

## Inspect First

Common existing manifest tools to check:

- `asset.find`
- `asset.guid_to_path`
- `asset.path_to_guid`
- `asset.read_text`
- `prefab.export_structure_json`
- `prefab.find_asset_references`
- `prefab.nested_prefab_overrides`
- `prefab.create_from_json`

Typical inspection sequence:

1. Confirm target prefab path.
2. Export or inspect structure with existing prefab tools.
3. Query dependencies or asset references if resource wiring matters.
4. Only add a new prefab-editing tool when the existing read tools prove insufficient.

## Creation and Editing Rules

- Use `prefab.create_from_json` when JSON creation is enough.
- If the task is structural prefab editing that existing tools do not cover, add the smallest prefab-focused generated tool.
- Avoid direct `.prefab` YAML editing for Unity tasks.
- Before changing the hierarchy of a nested prefab instance, inspect it with `prefab.nested_prefab_overrides`.
- Do not directly reparent children that still belong to a nested prefab instance. If hierarchy normalization is needed, prefer:
  - normalizing the source prefab first and then reinstalling or re-instantiating the nested prefab
  - unpacking the nested prefab only when the user explicitly approves that tradeoff
- Keep generated prefab tools scoped to one clear job, such as:
  - embedding a source prefab
  - rebinding a specific view component
  - standardizing one popup hierarchy

## Nested Prefab Recovery Pattern

When a generated tool fails with errors such as "Setting the parent of a transform which resides in a Prefab instance is not possible":

1. Stop retrying the same instance-side reparent operation.
2. Confirm the nested instance status with `prefab.nested_prefab_overrides`.
3. Move the hierarchy normalization to the source prefab instead of the nested instance.
4. Reinstall the normalized source prefab into the target prefab.
5. Re-run prefab structure export and binding validation.

## Typical Validation

After prefab writes, prefer Unity-side validation:

1. `compile.status`
2. Reload manifest if a tool was added or changed
3. `prefab.export_structure_json` to verify resulting hierarchy
4. `prefab.find_asset_references` or `prefab.nested_prefab_overrides` if the task involved references or nested prefabs

## Capability Gap Guidance

Add a new prefab tool only when the manifest cannot already do the required work through:

- create
- inspect
- reference lookup
- nested override analysis

If one step is missing, add only that step.
