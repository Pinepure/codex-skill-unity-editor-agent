# Scene Playbook

Use this file for scene object creation and scene-side inspection.

## Common Existing Tools

- `scene.create_empty`
- `scene.create_primitive`
- `scene.find_objects`
- `scene.find_asset_usages`
- `selection.get`
- `scene.save_open_scenes`

## Standard Sequence

1. Confirm compile status and manifest.
2. Use scene creation tools for simple object creation.
3. Use scene inspection tools to find objects or resource usages before adding custom scene tools.
4. Add a new scene tool only when the needed scene mutation or inspection is not already expressible through the manifest.

## Scene Rules

- Prefer scene tools over asking the user to perform manual editor actions.
- Respect confirmation requirements for save operations.
- Keep custom scene tools narrow and reversible where possible.

## Validation

- Re-query created objects with `scene.find_objects`
- Re-check compile state if any script/tool changed
- Save scenes only when required and when confirmation rules allow it
