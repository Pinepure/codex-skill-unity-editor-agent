# Pitfalls Log

Use this file when the Unity task resembles prior failures or recovery work. It records concrete mistakes, not just general advice, so the same dead ends are less likely to repeat.

## 1. Nested Prefab Hierarchy Surgery

### Failure Pattern

- A generated tool tries to reparent, regroup, or restructure children that still belong to a nested prefab instance inside another prefab.
- Unity throws errors such as `Setting the parent of a transform which resides in a Prefab instance is not possible`.
- `compile.status.hasCompileErrors` may become `true` even though C# compilation itself is fine, because the console now contains runtime errors from the failed tool call.

### Correct Recovery

1. Stop retrying the same instance-side reparent operation.
2. Inspect the target with `prefab.nested_prefab_overrides`.
3. If the subtree is a nested prefab instance, do one of these:
   - normalize the source prefab hierarchy first, then reinstall or re-instantiate it into the parent prefab
   - unpack the nested prefab only if the user explicitly accepts that tradeoff
4. Re-export the prefab structure.
5. Re-check binding paths and nested prefab overrides after reinstall.

### Default Rule

- Do not treat nested prefab instances like regular mutable hierarchies.
- Prefer source-prefab normalization over instance-side surgery.

## 2. Console Errors Are Not Always Compile Errors

### Failure Pattern

- `compile.status.hasCompileErrors` is `true`.
- The assumption is that the generated tool script failed to compile.
- The real issue is often a runtime error emitted by the tool after it successfully compiled.

### Correct Recovery

1. Check `compile.wait_until_idle` or `compile.status`.
2. Inspect `compile.errors` or `compile.errors_since_last_compile`.
3. Distinguish:
   - compiler errors
   - runtime console errors caused by the last tool call
4. Use `service.call_recent` and `service.log_recent` when the tool compiled but the operation failed.

### Default Rule

- Never infer the failure class from `hasCompileErrors` alone.

## 3. Generated UI Resources Need Importer Normalization

### Failure Pattern

- UI textures are moved out of generated folders such as `VectoUI`.
- The assets still import with tight mesh or generator defaults that are fine for some sprites but wrong for full-screen panels, dimmers, masks, or rectangular popup art.
- The popup renders or batches suboptimally, or later editing assumes the art is already normalized when it is not.

### Correct Recovery

1. Confirm which textures are actually used by the target prefab.
2. Move the textures into the feature-owned folder.
3. Normalize importer settings:
   - `textureType = Sprite`
   - `spriteMode = Single` when appropriate
   - `spriteMeshType = FullRect` for full-screen, dimmer, panel, and mask art
4. Create or update a `SpriteAtlas` for the feature set.
5. Validate the new asset paths from the consuming prefab side.

### Default Rule

- Do not move UI textures without also deciding whether importer settings and atlas layout must be normalized in the same task.

## 4. Validate Moved Assets From the Consumer Side

### Failure Pattern

- Assets are moved and the assumption is that references are now correct because the filenames look right.
- The real consumers or resolved paths are never verified.

### Correct Recovery

1. Before moving art, inspect the consuming prefab with:
   - `asset.dependencies`
   - `prefab.find_asset_references`
2. After moving art, verify the consuming prefab again.
3. Confirm importer state with `asset.meta_info` when the task is importer-sensitive.

### Default Rule

- Asset moves are incomplete until the consuming prefab proves that the new references are live.

## 5. Non-ASCII Asset Paths Need Safer JSON Invocation

### Failure Pattern

- A tool call is assembled manually in shell JSON with asset paths that contain Chinese or other non-ASCII characters.
- The request fails or appears to point at an invalid prefab even though the asset exists.

### Correct Recovery

1. Prefer a real JSON serializer or structured client when invoking tool calls for non-ASCII paths.
2. If shell usage is unavoidable, verify the exact encoded payload before blaming the Unity asset.
3. Retry with a safer transport before concluding that the prefab path is invalid.

### Default Rule

- For non-ASCII Unity asset paths, prefer Python, a structured HTTP client, or a tool wrapper over ad hoc shell-embedded JSON.

## 6. Temporary Generated Tools Must Not Accumulate

### Failure Pattern

- Recovery or migration work adds one-off generated tools.
- The tools remain in `Assets/Editor/AiUnityEditorAgent/GeneratedTools/` even after the prefab or asset state has been baked.

### Correct Recovery

1. Classify each generated tool after the task:
   - reusable now
   - task-specific
   - promote later
2. Delete task-specific tools with `tool.delete_generated`.
3. Preserve the lesson in the skill docs instead of preserving the disposable tool by default.

### Default Rule

- Prefer documenting the recovery pattern over keeping a one-off generated tool alive.

## 7. Temporary Project Editor Helpers Also Need Cleanup

### Failure Pattern

- Project-specific editor helpers are added only to stamp serialized references or run one-time binding.
- The helper and its editor-only binding methods remain in the project after the prefab data is already baked.

### Correct Recovery

1. Confirm the serialized references are already stored on the prefab or asset.
2. Remove editor-only custom inspectors or binding utilities that no longer provide an ongoing workflow.
3. Remove now-dead `#if UNITY_EDITOR` helper methods that only existed for that utility.
4. Re-run compile validation.

### Default Rule

- Do not keep maintenance burden for one-time binding helpers once the serialized state is stable.

## 8. Unity Asset Presence Does Not Imply Git Ownership

### Failure Pattern

- A Unity asset exists and resolves correctly through AssetDatabase.
- The assumption is that it belongs to the current git repository and should be committed or pushed from that repo.
- The asset actually lives in an untracked or externally generated folder.

### Correct Recovery

1. Treat Unity asset validity and git tracking as separate checks.
2. Before planning commits, verify git ownership of moved or generated assets.
3. If the asset belongs to a different repository or local-only workspace, update the correct repo or document the boundary clearly.

### Default Rule

- Never infer git scope from Unity scope.
