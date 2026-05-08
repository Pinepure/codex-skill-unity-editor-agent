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

## Validation

Validate asset-side results with a second Unity tool whenever practical:

- `asset.find` after creation
- `asset.meta_info` after import-sensitive changes
- dependency or reverse-dependency tools after wiring changes
