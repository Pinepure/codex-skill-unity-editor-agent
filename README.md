# unity-editor-agent

`unity-editor-agent` 是一个面向 Codex 的 skill，用来通过本地 AI Unity Editor Agent 服务完成 Unity 项目中的查询、修改、扩展和验证工作。

它的核心目标是让 AI 在 Unity 开发里遵循一套严格的 `Unity tool first` 工作流。连接前先读 `Library/AiUnityEditorAgent/endpoint.json` 和 `token.txt`，不要假设固定端口：

- 先读 `/health`，再读 `GET /manifest`，以当前 manifest 实际返回的工具列表为准
- 如果当前服务版本支持 `/manifest/search`、bundle 和 `/tool/describe_many`，按需使用；如果不支持，直接退回 `GET /manifest`
- 对非简单重复任务，先查 `workflow.match_recipes`，优先复用成功 workflow
- 当结果过大时，通过 `/result/{handleId}` 分页或分块继续读取
- 能用现有 Unity tool 完成的事情，就不用 shell 编辑或直接改 Unity 资产文件
- 只有在当前 manifest 明确证明能力缺失时，才新增 Unity 能力，而且要补最小能力，不要停在“没有现成工具”
- 多步任务优先 `batch.run` / `workset.*`，减少往返和 token 消耗
- 写操作前先 `validator.capture_baseline`，写完后必须 `validator.run`
- 看 `validator.run` 时要同时看 `checkedFiles`、`unstagedFiles`、`stagedFiles`、`untrackedFiles`
- 可复用的成功任务，结束前要 `workflow.record_success`
- 默认只做增量新增，不主动重构或合并已有项目内能力
- 不修改 `Packages/`
- 当前常见服务版本需要 `X-Unity-Ai-Token`，不要误用 Bearer 头
- Prefab 的对象引用绑定必须通过 Unity Editor 侧赋值并保存，不要直接改 `.prefab` YAML 试图写引用
- 一次性生成工具执行成功后应删除，把经验沉淀进 skill 文档，而不是长期堆在项目里

## 依赖

这个 skill 依赖 Unity 侧能力库：

- [Pinepure/com.aiunity.editor-agent](https://github.com/Pinepure/com.aiunity.editor-agent)

这个库负责在 Unity 项目里提供本地 AI 服务、轻量 manifest discovery、`/call/{toolId}` 调用面、结果分页句柄，以及 AI 可扩展的 Unity tool 机制。

根据该库的基本安装方式，Unity 侧可以按以下流程接入：

1. 下载并解压 `com.aiunity.editor-agent`
2. 在 Unity 中打开 `Window > Package Manager`
3. 点击 `+ > Add package from disk...`
4. 选择 `com.aiunity.editor-agent/package.json`
5. 打开 `Tools > AI Editor Agent > Control Center`

## 安装这个 Skill

把本仓库 clone 或复制到本地 Codex skills 目录：

```bash
${CODEX_HOME:-$HOME/.codex}/skills/unity-editor-agent
```

示例：

```bash
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills"
git clone git@github.com:Pinepure/codex-skill-unity-editor-agent.git "${CODEX_HOME:-$HOME/.codex}/skills/unity-editor-agent"
```

## 如何使用

在 Codex 对话里显式调用：

```text
Use $unity-editor-agent to inspect, modify, and extend a Unity project through the AI Unity Editor Agent protocol.
```

也可以用中文描述你的 Unity 任务，并显式带上 `$unity-editor-agent`。

适用场景包括：

- 查询 Unity 编译状态、资源、Prefab、Scene、已生成工具
- 通过 manifest 中已有的 Unity tool 完成 Prefab / Scene / Asset 操作
- 当现有能力全部不满足时，为项目新增最小化的 Unity tool
- 对 Unity 侧结果进行闭环验证

## Skill 内容

- `SKILL.md`：skill 触发描述与核心工作流
- `agents/openai.yaml`：Codex UI 元数据
- `references/ai-unity-editor-agent-protocol.md`：完整协议与生成工具约束
- `references/decision-policy.md`：决策规则与 tool-first 策略
- `references/capability-iteration.md`：能力缺口分析与智能迭代闭环
- `references/prefab-playbook.md`：Prefab 任务指南
- `references/asset-playbook.md`：Asset 任务指南
- `references/scene-playbook.md`：Scene 任务指南
- `references/diagnostics-playbook.md`：编译与工具故障诊断
- `references/validation-matrix.md`：最小验证规则

## 分享方式

最简单的分享方式，是把这个文件夹作为一个 Git 仓库公开，然后让其他人直接 clone 到自己的 Codex skills 目录：

```bash
git clone git@github.com:Pinepure/codex-skill-unity-editor-agent.git "${CODEX_HOME:-$HOME/.codex}/skills/unity-editor-agent"
```

## 说明

- 这个 skill 当前是“工作流约束 + 参考文档”型 skill，不自带执行脚本
- 它默认假设目标 Unity 项目已经安装并运行了 `com.aiunity.editor-agent`
- 它的重点不是绕开 Unity，而是让 AI 优先通过 Unity 自身暴露出来的工具面完成开发和验证
