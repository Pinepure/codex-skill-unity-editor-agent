# Diagnostics Playbook

Use this file when compile status, import state, or tool execution does not match expectations.

## First Checks

1. `compile.status`
2. `compile.errors` when `hasCompileErrors == true`
3. `compile.wait_until_idle` when Unity may still be compiling or importing
4. `service.log_recent` when a tool fails unexpectedly
5. `service.call_recent` when you need the recent invocation history

## After `tool.upsert_script`

Always do this sequence:

1. Wait until compile is idle
2. Check compile errors
3. Reload manifest
4. Confirm the expected tool ID appears
5. Then call the tool

## Failure Patterns

### Tool ID missing after install

- Reload the manifest
- Check compile status
- Check compile errors
- Check service logs

### Asset exists on disk but Unity disagrees

- Refresh assets if there is a concrete import reason
- Re-check manifest tools for import diagnostics
- If the manifest still lacks a sufficient diagnostic path, add a narrow import-state tool

### Tool ran but result is wrong

- Prefer validating assumptions with existing read tools
- Fix the generated tool narrowly
- Reinstall, recompile, reload manifest, and re-run

Do not guess around failures when diagnostics are available.
