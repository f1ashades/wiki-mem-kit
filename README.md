# wiki-mem-kit

> 给 AI agent 用户的业务记忆系统：把会议、决策、需求沉淀为 markdown，
> 让你的 Claude Code / Cursor / Codex 在写代码或脑暴时拿回业务上下文。

## 这是给谁的

如果你在做以下事情：

- 用 AI agent (Claude Code / Cursor / Codex 等) 写代码或做产品脑暴
- 经常需要回顾"我之前和某商家 / 同事 / 客户聊过什么"
- 苦于 agent 没有你的业务上下文，每次都要从头解释一遍

wiki-mem-kit 帮你把这些上下文持续沉淀成 markdown 笔记。当前 MVP 阶段，agent 通过**按目录读取**（如 `decisions/`、`meetings/`）拿回上下文；语义召回 / 自动相关性匹配是 MCP 完成后的目标。

## 核心理念

- **数据 = markdown + git**，你完全可控、可移植、可审计
- **写入 = AI Skill**，agent 按预设流程把原始材料沉淀成结构化 page
- **读取（当前 MVP）= 目录读取**，agent 直接读 `decisions/*.md`、`meetings/*.md` 等子目录
- **读取（future）= MCP server**（开发中），agent 通过工具调用做语义召回，不需要全量加载
- **agent 通用**：优先 Claude Code，其次 Cursor 和 Codex，其他 agent 走 fallback 也能用

## 怎么开始用

**不用 clone，不用手动配置。**

打开你的 AI agent，对它说：

```
帮我装 wiki-mem-kit，github.com/masaka/wiki-mem-kit
```

agent 会自动完成：
- 拉取本仓库到本地
- 读取 `setup/SKILL.md` 走安装流程
- 问你 4 个偏好问题（数据放哪、隐私模式、是否迁移现有笔记等）
- 创建你的私有数据目录
- 配置 agent 的 skill 加载与未来的 MCP 接入
- 输出 next steps

整个过程 2-3 分钟，全程对话式。

**支持的 agent**：

| Agent | 支持级别 |
|-------|---------|
| Claude Code | Tier 1 - 最佳体验 |
| Cursor | Tier 2 - 完整支持 |
| Codex (OpenAI) | Tier 2 - 完整支持 |
| Gemini CLI / Aider / Cline / 其他 | Tier 3 - 通用 fallback |

详见 [docs/agent-compat.md](docs/agent-compat.md)。

## 你的数据放在哪 / 隐私

完全在你本地。安装时 agent 会问你"数据目录位置"，默认 `~/business-memory/`，可自定义。

如果你选「本地模式」（不推送任何 git remote），数据永远不出你这台机器。
如果你选「私有 git remote」，你自行决定推到哪——公司内 GitLab / 你私有的 GitHub 仓库 / Gitea / 等等。

**wiki-mem-kit 本身不收集任何数据，没有 telemetry**。

## 它生成什么样的 page

每个 page 都是 markdown，结构如下：

```markdown
---
title: 棋牌室硬件状态接入方案
type: decision
schema_version: 1
sources:
  - raw/2026-04-25-王五会议.md
status: active
review_status: needs_review
---

## 当前结论

v1 范围：仅接入灯光控制 + 门禁状态
v2 留空：计费系统暂不集成

## 待确认事实

- 预算 3 万是否是对外承诺，待用户确认。

## 演化时间线

- 2026-04-25 王五同意先做硬件状态板 v1 范围 → [[raw/2026-04-25-王五会议]]
```

- 顶部 = 当前事实（compiled 区，AI 写代码时读这里）
- 待确认事实 = 金额、承诺、冲突等需要用户确认的信息；`review_status: needs_review` 时读取侧不能当硬约束
- 底部 = 时间线（不删除历史，可追溯演化）
- frontmatter 的 `sources` 自动建立 page 之间的图谱关联

## 仓库结构

```
wiki-mem-kit/
├── README.md                 # 本文档（给人看）
├── setup/
│   └── SKILL.md              # 安装流程（给 agent 看）
├── health/
│   └── SKILL.md              # vault 体检流程（给 agent 看）
├── presets/
│   └── professional-work/    # 工作场景预设
│       ├── SKILL.md          # sink 流程
│       ├── decision.yaml     # 决策类型定义
│       └── meeting.yaml      # 会议类型定义
├── core/                     # MCP server (开发中)
└── docs/
    ├── page-type-spec.md     # page_type 规范
    └── agent-compat.md       # agent 兼容性宪法
```

## MVP 范围

- ✅ professional-work 预设：decision + meeting 两个 page_type
- ✅ Claude Code / Cursor / Codex 三个 agent 的安装支持
- ⏳ MCP server（读取侧 5-7 个工具）
- ⏳ 更多 preset（personal-life、reading-notes、merchant-profile 等）
- ⏳ Plugin marketplace 一键安装

## 设计哲学

借鉴自三个市场验证过的项目：

- **[basic-memory](https://github.com/basicmachines-co/basic-memory)** — 提供了 Entity / Observation / Relation 三元组 + folder=type 的简洁 schema
- **[gbrain](https://github.com/garrytan/gbrain)** — 提供了「单文件 = compiled truth + timeline 双区」的核心范式 + thin harness fat skills 的 MCP 设计
- **[llm_wiki](https://github.com/nashsu/llm_wiki)** — 提供了 source 自动关联的图谱权重思路 + 二步式 ingest

我们的差异化：

- 三者只有 basic-memory 有 MCP，其他靠 GUI / CLI；wiki-mem-kit 走 MCP-first
- 三者都偏向「单一 agent 客户端」；wiki-mem-kit 显式做 agent 兼容分层
- 三者都偏向「整套技术栈大」；wiki-mem-kit 走「薄配置 + 厚 SKILL」极简路线

详细对比见 [docs/page-type-spec.md](docs/page-type-spec.md) 和 [docs/agent-compat.md](docs/agent-compat.md)。

## License

MIT (待补充正式 LICENSE 文件)

## 反馈与贡献

issue / PR welcome。当前是 MVP 阶段，欢迎用着用着提改进建议。
