# Validation Matrix

Use this file to choose a minimum verification set before closing Unity work.

## Tool Installation or Tool Changes

- Required checks:
  - `compile.status`
  - `compile.snapshot` if compile failed
  - re-check `manifestHash` and refresh discovery
  - confirm expected tool ID is present

## Prefab Writes

- Required checks:
  - compile state
  - prefab inspection tool such as `prefab.export_structure_json`
- Optional checks:
  - `prefab.find_asset_references`
  - `prefab.nested_prefab_overrides`
  - `asset.dependencies`

## Asset Wiring Changes

- Required checks:
  - compile state
  - an asset inspection or dependency tool
- Optional checks:
  - `asset.meta_info`
  - `asset.reverse_dependencies`
  - `asset.reference_chain`
  - `prefab.find_asset_references` when moved textures, sprites, or atlases are consumed by prefabs

## Scene Changes

- Required checks:
  - compile state if any scripts changed
  - a scene inspection tool such as `scene.find_objects`
- Optional checks:
  - `scene.find_asset_usages`
  - scene save status if the task required saving

## Diagnostics Tasks

- Required checks:
  - `compile.status`
  - `compile.snapshot` or `compile.errors_summary` when relevant
  - `service.log_recent` when a tool fails
- Optional checks:
  - `service.call_recent`
  - asset refresh and re-check, only when import state is relevant

## Nested Prefab Hierarchy Repairs

- Required checks:
  - compile state
  - `prefab.nested_prefab_overrides` before and or after the repair when nesting behavior matters
  - `prefab.export_structure_json` after the repair
- Optional checks:
  - `prefab.find_asset_references`
  - binding or component-path verification through the project-specific prefab inspection output
