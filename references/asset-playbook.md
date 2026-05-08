# Asset Playbook

Use this file for asset lookup, GUID/path conversion, dependency analysis, reverse reference analysis, text asset reads, and import-state diagnosis.

## Common Existing Tools

- `asset.find`
- `asset.dependencies`
- `asset.reverse_dependencies`
- `asset.reverse_dependencies_batch`
- `asset.reverse_dependencies_direct_batch`
- `asset.reference_chain`
- `asset.guid_to_path`
- `asset.path_to_guid`
- `asset.read_text`
- `asset.meta_info`
- `asset.refresh`

## Standard Sequence

1. Use `asset.find` to locate candidates.
2. Convert GUIDs and paths with the dedicated conversion tools instead of guessing.
3. Use dependency or reverse-dependency tools before inventing a custom search path.
4. Use `asset.read_text` only for text assets under `Assets/`.
5. If disk state and Unity state disagree, use an import-state diagnostic tool if the manifest provides one; otherwise prove the gap before adding it.

## Asset Rules

- Do not search the filesystem directly for answers that AssetDatabase-backed tools already provide.
- Normalize asset paths and keep them under `Assets/` unless a manifest tool explicitly supports `Packages/`.
- Use `asset.refresh` only when there is a concrete reason to sync Unity’s asset database with disk.

## UI Asset Migration Pattern

When moving UI textures from generated folders into a feature-owned location:

1. Confirm the real consumers with prefab dependency or reference tools before moving anything.
2. Move the textures into the feature folder.
3. Verify or correct importer state:
   - `textureType = Sprite`
   - `spriteMode = Single` when the source is a standalone sprite
   - `spriteMeshType = FullRect` for full-screen panels, dimmers, masks, or other rectangular UI art that should not use a tight mesh
4. Create or update a `SpriteAtlas` when the moved textures belong to one popup or feature set.
5. Validate that the prefab references now resolve to the new asset paths.

## Validation

Validate asset-side results with a second Unity tool whenever practical:

- `asset.find` after creation
- `asset.meta_info` after import-sensitive changes
- dependency or reverse-dependency tools after wiring changes
- `prefab.find_asset_references` after moving art that is consumed by a prefab
