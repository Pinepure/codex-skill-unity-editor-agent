# Diagnostics Playbook

Use this file when compile status, import state, or tool execution does not match expectations.

## First Checks

1. `compile.status`
2. If the service exposes richer compile-summary tools, use them.
3. Otherwise use `compile.errors` and `compile.errors_since_last_compile` when available.
4. Use paged `compile.errors` when full entries are required
5. `service.log_recent` when a tool fails unexpectedly
6. `service.call_recent` when you need the recent invocation history

## After `tool.upsert_script`

Always do this sequence:

1. Wait until compile is idle
2. Check the best compile diagnostics that the current service actually exposes
3. Re-read `GET /manifest` or equivalent live discovery
4. Confirm the expected tool ID appears
5. Then call the tool

## Failure Patterns

### Tool ID missing after install

- Reload the manifest
- Check compile status
- Check available compile diagnostics
- Inspect the current Unity compile/console output before assuming a manifest cache issue. If a generated tool never appears, the script usually failed to compile and never entered the Editor assembly.
- When summaries look incomplete, page through `compile.errors` or inspect the live Editor output so you do not miss generated-tool compile errors.
- Check service logs

### Asset exists on disk but Unity disagrees

- Refresh assets if there is a concrete import reason
- Re-check manifest tools for import diagnostics
- If the manifest still lacks a sufficient diagnostic path, add a narrow import-state tool
- For prefab field references, prefer a small Editor binding tool over direct YAML inspection or manual GUID/fileID guessing

### Tool ran but result is wrong

- Prefer validating assumptions with existing read tools
- Fix the generated tool narrowly
- Reinstall, recompile, reload manifest, and re-run

Do not guess around failures when diagnostics are available.
