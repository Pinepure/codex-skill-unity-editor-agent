# unity-editor-agent

`unity-editor-agent` 是一个面向 Codex 的 skill，用来通过本地 AI Unity Editor Agent 服务完成 Unity 项目中的查询、修改、扩展和验证工作。

它的核心目标是让 AI 在 Unity 开发里遵循一套严格的 `Unity tool first` 工作流：

- 把 Unity 的实时 `/manifest` 视为唯一事实源
- 能用现有 Unity tool 完成的事情，就不用 shell 编辑或直接改 Unity 资产文件
- 只有在当前 manifest 明确证明能力缺失时，才新增 Unity 能力
- 默认只做增量新增，不主动重构或合并已有项目内能力
- 不修改 `Packages/`

## 依赖

这个 skill 依赖 Unity 侧能力库：

- [Pinepure/com.aiunity.editor-agent](https://github.com/Pinepure/com.aiunity.editor-agent)

这个库负责在 Unity 项目里提供本地 AI 服务、`/manifest`、`/call/{toolId}` 调用面，以及 AI 可扩展的 Unity tool 机制。

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
git clone <你的仓库地址> "${CODEX_HOME:-$HOME/.codex}/skills/unity-editor-agent"
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
git clone <你的仓库地址> "${CODEX_HOME:-$HOME/.codex}/skills/unity-editor-agent"
```

## 说明

- 这个 skill 当前是“工作流约束 + 参考文档”型 skill，不自带执行脚本
- 它默认假设目标 Unity 项目已经安装并运行了 `com.aiunity.editor-agent`
- 它的重点不是绕开 Unity，而是让 AI 优先通过 Unity 自身暴露出来的工具面完成开发和验证
