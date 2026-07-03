# AI Unity Editor Agent Protocol

This document is the built-in operating manual for AI agents that control Unity through the AI Unity Editor Agent package.

Treat the live service surface as authoritative. Newer builds may expose `manifestHash`, manifest search, bundles, and richer compile-summary endpoints; older local builds such as `0.1.1` may not. The workflow must degrade cleanly to `/health`, `GET /manifest`, `POST /call/{toolId}`, available result handles, and the diagnostics tools that actually exist.

## 1. Connection

Default base URL:

```text
Read <ProjectRoot>/Library/AiUnityEditorAgent/endpoint.json for the current host and port.
```

Default token file:

```text
<ProjectRoot>/Library/AiUnityEditorAgent/token.txt
<ProjectRoot>/Library/AiUnityEditorAgent/endpoint.json
```

Required request header unless disabled by the user in the Control Center:

```http
X-Unity-Ai-Token: <token>
```

Health and discovery metadata:

```http
GET /health
```

Lightweight manifest summary:

```http
GET /manifest
X-Unity-Ai-Token: <token>
```

Full manifest fallback when supported:

```http
GET /manifest/full
X-Unity-Ai-Token: <token>
```

Manifest search when supported:

```http
POST /manifest/search
X-Unity-Ai-Token: <token>
Content-Type: application/json

{"query":"find prefab dependencies","limit":6}
```

Focused capability bundles when supported:

```http
GET /manifest/bundles
GET /manifest/bundle/asset-analysis
X-Unity-Ai-Token: <token>
```

Describe exact tool schemas when supported:

```http
POST /tool/describe_many
X-Unity-Ai-Token: <token>
Content-Type: application/json

{"ids":["asset.find","asset.dependencies"]}
```

Call a tool:

```http
POST /call/{toolId}
X-Unity-Ai-Token: <token>
Content-Type: application/json

{...tool arguments...}
```

Read next pages or text chunks from a result handle:

```http
GET /result/{handleId}?offset=0&limit=20
X-Unity-Ai-Token: <token>
```

Read the concise brief:

```http
GET /agent/brief
X-Unity-Ai-Token: <token>
```

Read the full manual:

```http
GET /agent
X-Unity-Ai-Token: <token>
```

## 2. Required AI workflow

For every Unity task:

1. Call `GET /health`.
2. Read `GET /manifest` and treat the returned tool list as the current source of truth.
3. If the service exposes `manifestHash`, search, bundles, or describe-many, use them. If not, continue from the manifest you have instead of blocking.
4. Prefer narrower discovery endpoints when they exist.
5. Call exact schema endpoints only when the current service actually exposes them.
6. For non-trivial or repeated goals, query `workflow.match_recipes` before planning from scratch.
7. Call the selected tool through `POST /call/{toolId}`.
8. If a tool returns `resultHandle`, page additional data through `GET /result/{handleId}` instead of re-running the tool with larger limits.
9. Use `GET /manifest/full` only when that endpoint exists and the lighter discovery routes are insufficient.
10. If no suitable tool exists, generate a new Editor tool script with `tool.upsert_script`, wait for compile completion, then repeat discovery.
11. After validated reusable success, record the flow with `workflow.record_success`.

## 3. Protocol priorities

- Prefer `GET /health` over repeatedly loading manifests.
- Prefer `GET /manifest` over `GET /manifest/full`.
- Prefer `POST /manifest/search` over scanning all tools mentally when the endpoint exists.
- Prefer `POST /tool/describe_many` over reading schemas for tools you will not call when the endpoint exists.
- Prefer `GET /result/{handleId}` over asking a tool to return larger and larger payloads.
- Prefer the best compile diagnostics the current service actually exposes before requesting verbose console entries.

## 4. Tool method implementation contract

All AI-generated tools must follow this C# shape:

```csharp
#if UNITY_EDITOR
using System;
using UnityEditor;
using UnityEngine;
using AiUnity.EditorAgent;

namespace AiUnity.EditorAgent.Generated
{
    internal static class MyGeneratedTools
    {
        [Serializable]
        private sealed class Args
        {
            public string name;
        }

        [AiTool(
            "example.say_hello",
            "Returns a greeting.",
            @"{""type"":""object"",""properties"":{""name"":{""type"":""string""}}}",
            @"{""type"":""object"",""properties"":{""message"":{""type"":""string""}}}"
        )]
        public static string SayHello(string argsJson)
        {
            Args args = JsonUtility.FromJson<Args>(argsJson);
            string name = args == null || string.IsNullOrEmpty(args.name) ? "Unity" : args.name;
            return @"{""message"":" + AiJson.Quote("Hello, " + name) + "}";
        }
    }
}
#endif
```

Rules:

- Method must be `static`.
- Method must return `string`.
- Method must accept exactly one argument: `string argsJson`.
- Method must return a valid JSON value, usually a JSON object.
- Method must have `[AiTool]`.
- Tool IDs must be unique and stable, using dotted names such as `scene.create_empty` or `asset.reverse_dependencies`.
- Use `JsonUtility.FromJson<T>()` for simple arguments.
- Use `AiJson.Quote()` and `AiJson.Escape()` when composing JSON manually.
- High-risk tools must set `Danger = AiToolDanger.High` and `RequiresConfirmation = true`.
- Generated tools should use the namespace `AiUnity.EditorAgent.Generated`.
- Generated scripts must be installed through `tool.upsert_script`; do not write outside `Assets/Editor/AiUnityEditorAgent/GeneratedTools/`.

## 5. Manifest conventions

`GET /manifest` may return a lightweight summary or a fuller manifest, depending on service version. Newer builds may look like:

```json
{
  "protocolVersion": "2.0",
  "serviceVersion": "0.3.0",
  "manifestHash": "7c3d...",
  "toolCount": 35,
  "namespaces": [
    { "id": "asset", "count": 7 }
  ],
  "tools": [
    {
      "id": "asset.find",
      "namespaceId": "asset",
      "description": "Searches project assets...",
      "danger": "low",
      "requiresConfirmation": false
    }
  ]
}
```

Older builds may skip `manifestHash` entirely and still be valid. In those cases, re-read `GET /manifest` after tool installation or service changes instead of pretending a cache key exists.

`GET /manifest/full` returns the full fallback manifest with `argsSchemaJson`, `returnSchemaJson`, and source metadata when supported.

`POST /tool/describe_many` returns exact schemas for the requested tool ids when supported:

```json
{
  "manifestHash": "7c3d...",
  "requestedCount": 2,
  "returnedCount": 2,
  "missingIds": [],
  "tools": [
    {
      "id": "asset.find",
      "namespaceId": "asset",
      "argsSchemaJson": "{...}",
      "returnSchemaJson": "{...}"
    }
  ]
}
```

Only load full schemas for tools you actually plan to call. If the route is unavailable, rely on the manifest fields the service does provide.

## 6. Search and bundle conventions

Use manifest search for open-ended tasks when that route exists:

```json
{
  "query": "inspect prefab dependencies",
  "limit": 6
}
```

Use bundle loading for focused workflows when the current service exposes bundle routes:

- `asset-analysis`
- `scene-editing`
- `prefab-authoring`
- `tool-authoring`
- `service-diagnostics`

Search results include `score` and `whyMatched` when supported. If search is unavailable, shortlist from the manifest directly.

## 7. Result handle conventions

Large tool responses may return this shape:

```json
{
  "summary": {
    "source": "asset.find",
    "totalFound": 183
  },
  "returned": 20,
  "pageSize": 20,
  "total": 183,
  "hasMore": true,
  "resultHandle": "abcd1234",
  "items": []
}
```

When `resultHandle` is present, continue with:

```http
GET /result/abcd1234?offset=20&limit=20
```

Text readers such as `asset.read_text` may return chunked `content` plus `resultHandle`.

`batch.run` placeholder paths must target the wrapped step result shape. Typical examples:

1. `{{step1.result.result.path}}`
2. `{{step2.result.result.resultHandle}}`
3. `{{step3.ok}}`

## 8. Built-in high-value tools

Common tools available in a fresh install vary by service version. A newer install may include:

- `system.health`
- `manifest.get`
- `manifest.get_summary`
- `manifest.search`
- `manifest.list_bundles`
- `manifest.get_bundle`
- `tool.describe_many`
- `agent.get_brief`
- `agent.get_manual`
- `compile.status`
- `compile.snapshot`
- `compile.errors`
- `compile.errors_summary`
- `console.clear`
- `asset.refresh`
- `asset.find`
- `asset.dependencies`
- `asset.reverse_dependencies`
- `asset.guid_to_path`
- `asset.path_to_guid`
- `asset.read_text`
- `prefab.create_from_json`
- `selection.get`
- `scene.create_empty`
- `scene.create_primitive`
- `tool.upsert_script`
- `tool.list_generated`
- `tool.delete_generated`
- `tool.get_template`
- `service.log_recent`
- `service.call_recent`

## 9. Compile diagnostics

Prefer the richest compile diagnostics the current service exposes. Newer builds may support:

1. `compile.snapshot`
2. `compile.errors_summary`
3. `compile.errors` with `includeStackTrace=false`
4. `compile.errors` with `includeStackTrace=true` only when stack details are needed

Older builds may only expose:

1. `compile.status`
2. `compile.errors_since_last_compile`
3. `compile.errors`
4. `service.log_recent` and `service.call_recent`

After calling `tool.upsert_script`:

1. Poll `compile.status` until `isCompiling == false`.
2. If compile errors are present, call the best available diagnostics for the current service.
3. If the first diagnostic is insufficient, expand to paged `compile.errors` or service logs.
4. Resume discovery through `GET /health` and `GET /manifest`, plus richer discovery routes when available.

## 9.1 Validation scope

`validator.run` may return explicit scope fields such as:

- `checkedFiles`
- `unstagedFiles`
- `stagedFiles`
- `untrackedFiles`

Read those fields before deciding a task is complete, especially when Git state matters.

## 10. Security rules for AI agents

- Never request a tool call that is not present in the latest discovery results or full manifest.
- Never bypass `requiresConfirmation`.
- Never write files outside generated tool folders.
- Never create tools that execute shell commands unless the user explicitly approves and the tool is marked high risk.
- Never read or expose secrets from project files unless the user explicitly asks.
- Prefer small, single-purpose tools over large unrestricted tools.
- Include argument schemas and return schemas for every generated tool.
- After adding or changing tools, refresh discovery before assuming old capability caches still apply. Use `manifestHash` only when the service actually returns it.

## 11. Minimal curl examples

Read token:

```bash
TOKEN=$(cat Library/AiUnityEditorAgent/token.txt)
```

Health:

```bash
curl -H "X-Unity-Ai-Token: $TOKEN" <BASE_URL_FROM_ENDPOINT_JSON>/health
```

Search candidate tools when supported:

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "X-Unity-Ai-Token: $TOKEN" \
  -d '{"query":"find prefab dependencies","limit":6}' \
  <BASE_URL_FROM_ENDPOINT_JSON>/manifest/search
```

Describe exact schemas when supported:

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "X-Unity-Ai-Token: $TOKEN" \
  -d '{"ids":["asset.find","asset.dependencies"]}' \
  <BASE_URL_FROM_ENDPOINT_JSON>/tool/describe_many
```

Call asset search:

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "X-Unity-Ai-Token: $TOKEN" \
  -d '{"filter":"t:Prefab","folders":["Assets"],"maxResults":100,"pageSize":20}' \
  <BASE_URL_FROM_ENDPOINT_JSON>/call/asset.find
```

Read the next page from a result handle:

```bash
curl -H "X-Unity-Ai-Token: $TOKEN" \
  "<BASE_URL_FROM_ENDPOINT_JSON>/result/<HANDLE>?offset=20&limit=20"
```
