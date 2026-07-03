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

1. Check `compile.status`.
2. Inspect `compile.snapshot`, then `compile.errors_summary` or paged `compile.errors` when needed.
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

## 7. Prefab Object References Must Be Serialized by Unity

### Failure Pattern

- A prefab needs a sprite, prefab, or other `ObjectReference` field updated.
- The change is attempted by editing `.prefab` YAML directly or by guessing `guid` and `fileID`.
- Unity does not surface the new dependency, `prefab.find_asset_references` still shows nothing, or the field silently remains unset.

### Correct Recovery

1. Stop trying to stamp the reference through raw YAML.
2. Add or reuse a narrow Editor tool that:
   - loads prefab contents with `PrefabUtility.LoadPrefabContents`
   - loads the target assets with `AssetDatabase.LoadAssetAtPath`
   - assigns the component fields directly
   - saves through `PrefabUtility.SaveAsPrefabAsset`
3. Validate from the consumer side with `asset.dependencies` and or `prefab.find_asset_references`.
4. Delete the tool if it was only for one task.

### Default Rule

- Unity-managed serialized references should be written by Unity, not by hand-authored prefab YAML.

## 8. Temporary Project Editor Helpers Also Need Cleanup

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

## 9. Unity Asset Presence Does Not Imply Git Ownership

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

## 10. Service-Version Drift Is Normal

### Failure Pattern

- The skill docs assume `manifestHash`, `/manifest/search`, bundle endpoints, or `compile.snapshot` exist everywhere.
- The local service is an older build such as `0.1.1` and only exposes `GET /manifest`, `compile.status`, and a narrower tool surface.
- The agent stalls on non-existent endpoints instead of continuing with what is available.

### Correct Recovery

1. Start with `/health`.
2. Read the current manifest and trust the advertised tools over the docs.
3. If richer discovery or diagnostics endpoints are missing, fall back to `GET /manifest`, `compile.errors`, `compile.errors_since_last_compile`, and service logs.
4. Add a small generated tool when the missing capability is real.

### Default Rule

- The live service surface wins over static skill docs.

## 11. Image-Generated Sprite Art Needs a Chroma-Key-to-Alpha Pass

### Failure Pattern

- Raster character or VFX art is generated for Unity through a normal image model path.
- The raw output is dropped straight into the project with a flat green background still attached, or the `n x m` frame sheet is assembled before background removal.
- The imported sprite carries dirty edges, incorrect bounds, or bakes the key color into later animation work.

### Correct Recovery

1. Generate the art on a perfectly flat chroma-key background that is absent from the subject, typically `#00ff00`.
2. Remove the key color locally to produce alpha PNGs before Unity import.
3. For frame animation, clean each frame first, then assemble the final sheet only if a sheet is truly required.
4. If the animation is meant to use separate frame PNGs, import those frame PNGs individually as single sprites and bind the animator to those assets directly.

### Default Rule

- Do not treat image-model output as Unity-ready until the chroma key has been converted to alpha and the intended frame-packaging decision is explicit.

## 12. URP Particle Textures Need Transparent Surface Materials

### Failure Pattern

- A flame, ember, or firefly PNG has correct alpha.
- The particle material is still left on an opaque surface configuration.
- The scene shows black fringes, dark quads, or other alpha-looking artifacts even though the source texture itself is fine.

### Correct Recovery

1. Keep the particle texture import clean: alpha enabled, single sprite or texture as intended, no leftover sheet metadata when the asset is meant to be single-frame.
2. On the particle material, explicitly set:
   - `SurfaceType = Transparent`
   - transparent render queue / transparent render tag
   - alpha-friendly blend and `ZWrite = 0` when appropriate for the shader
3. Rebuild or re-save the material through Unity, then validate the resulting `.mat` contents or runtime behavior.

### Default Rule

- PNG alpha alone is insufficient for URP particle art; material transparency settings are part of the asset contract.
