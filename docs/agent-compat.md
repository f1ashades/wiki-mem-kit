# Agent 兼容性宪法

wiki-mem-kit 顶层设计原则：**尽量做到能支持各类 AI agent 通用，冲突较大时偏向 Claude Code，其次按使用量热度照顾。**

本文档定义三层分层结构。所有 SKILL.md 涉及 agent 特化的步骤都必须按本宪法执行。

## 三层定义

### Tier 1: Claude Code

Anthropic 自家产品，且 wiki-mem-kit 概念与 Claude Skill 体系最契合，享受最佳体验：

- 用 AskUserQuestion 做结构化输入（仅 Cowork 环境可用，普通 Claude Code 也用聊天）
- 通过 `.claude/skills/` symlink 自动加载 SKILL.md
- 写 `.mcp.json` (项目级) 或 `~/.claude/mcp_settings.json` (用户级)
- 未来支持 plugin marketplace 一键安装

### Tier 2: 单独照顾的高使用量 agent

当前列表（按字母序）：

- **Codex** (OpenAI Codex CLI)
- **Cursor**

每个 Tier 2 agent 在本文档下方有专属技术规约（skill 加载位置、MCP 配置位置等）。

### Tier 3: 通用 fallback

所有 Tier 1/2 之外的 agent（Gemini CLI、Aider、Cline、RooCode、Windsurf、Kiro 等）。

统一处理方式：

- 用聊天提问，不假设结构化输入
- 告诉用户每次手动让 agent 读 SKILL.md
- 打印 MCP 配置片段，让用户自己粘贴

体验会比较糙但能用。等到某个 Tier 3 agent 满足 Tier 2 入选标准时升级。

## Tier 2 入选标准

agent 必须**全部**满足以下三条才能进 Tier 2：

1. **有官方文档化的「长期指令」机制**（rules / skill / 系统提示文件），且格式相对稳定（不是月月大改）
2. **原生支持 MCP**，且配置位置可被脚本读写
3. **使用量足够大**：GitHub repo 超过 10k star，或主流技术媒体明显在讨论

每 6 个月评估一次 Tier 2 候选名单。

## Tier 升降规则

- Tier 3 → Tier 2：满足三条标准 + 有 wiki-mem-kit 用户实际反馈想用
- Tier 2 → Tier 3：失去使用量热度（媒体讨论显著减少 / 主项目半年无更新）
- 任何分级变动需在本文档「变更历史」段记录

## SKILL.md 中的分支语法

所有 SKILL.md 涉及 agent 特化的步骤，**必须**按以下结构写：

````markdown
## 第 N 步: <要做的事>

请你（agent）先识别自己是哪一类，按对应分支执行。

### 如果你是 Claude Code
<具体步骤>

### 如果你是 Cursor
<具体步骤>

### 如果你是 Codex
<具体步骤>

### 如果都不是
<fallback 步骤>
````

约束：

- 每个分支独立可读、可测、可单独失败
- 不要在分支之间共享状态或假设执行顺序
- fallback 分支必须存在，不能省略

## Agent 自识别流程

如果 agent 不确定自己是哪类，按以下顺序判断：

1. **检查环境标志**：
   - 工具列表中有 `mcp__cowork__*` 工具 → Claude Code in Cowork
   - 工具列表中有 `.cursor/` 路径或 Cursor-specific MCP → Cursor
   - 工具列表中有 `~/.codex/` 路径或 AGENTS.md 约定 → Codex
2. **问用户**："你正在用哪个 AI 工具？" 并给出 Tier 1/2 + "其他" 的选项
3. **默认**：如果用户也不清楚，按 Tier 3 fallback 处理

## 各 agent 技术规约

### Claude Code (Tier 1)

| 项目 | 位置 / 方式 |
|------|------|
| Skill 加载（项目级） | `.claude/skills/<skill-name>/SKILL.md` |
| Skill 加载（用户级） | `~/.claude/skills/<skill-name>/SKILL.md` |
| MCP 配置（项目级） | `.mcp.json` |
| MCP 配置（用户级） | `~/.claude/mcp_settings.json` |
| 结构化输入 | AskUserQuestion (Cowork 环境) / 普通聊天 |

### Cursor (Tier 2)

| 项目 | 位置 / 方式 |
|------|------|
| Skill 加载 | `.cursor/rules/<name>.mdc` (多文件) |
| MCP 配置（项目级） | `.cursor/mcp.json` |
| MCP 配置（用户级） | `~/.cursor/mcp.json` |
| 结构化输入 | 无，用聊天 |

#### Cursor `.mdc` 文件 frontmatter 模板

```yaml
---
description: 一句话描述，用户和 agent 都能看到
globs: ["raw/**/*.md", "decisions/**/*.md"]   # 可选，文件匹配触发
alwaysApply: false                              # true = 全局生效
---
```

### Codex (Tier 2)

| 项目 | 位置 / 方式 |
|------|------|
| Skill 加载（项目级） | `AGENTS.md` (在项目根，追加 markdown 章节) |
| Skill 加载（用户级） | `~/.codex/instructions.md` |
| MCP 配置 | `~/.codex/config.toml`，段名 `[mcp_servers.<name>]` |
| 结构化输入 | 无，用聊天 |

#### Codex MCP 配置片段模板

```toml
[mcp_servers.wiki-mem]
command = "wiki-mem-mcp"
args = ["--data-dir", "<data_dir>"]
```

## 变更历史

- 2026-04-26: 初版宪法。Tier 2 包含 Cursor 和 Codex。
