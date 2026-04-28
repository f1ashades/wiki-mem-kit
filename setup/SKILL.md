# wiki-mem-kit Setup Skill

本 SKILL 由 agent 在用户**首次安装** wiki-mem-kit 时执行。

## 触发条件

用户在 agent 中说类似：
- "帮我装 wiki-mem-kit"
- "set up wiki-mem-kit from github.com/masaka/wiki-mem-kit"
- "我想用 wiki-mem-kit 这个项目"

agent 应：
1. 把本仓库 clone 到本地（默认位置 `~/.local/share/wiki-mem-kit/`，可被用户覆盖）
2. 读取本 SKILL.md
3. 按以下流程执行

---

## 总流程概览

```
Step 1: 自识别 agent 类型
Step 2: 收集用户偏好（4 个问题）
Step 3: 创建数据目录 + 写 config.yaml + 初始化 git
Step 4: 加载 skill（按 agent 分支）
Step 5: 配置 MCP server（按 agent 分支，MVP 阶段跳过）
Step 6: 健康检查 + 输出 next steps
```

---

## Step 1: 自识别 agent

按 `docs/agent-compat.md` 的「Agent 自识别流程」执行。识别结果之一：
- `claude-code`
- `cursor`
- `codex`
- `unknown`（走 Tier 3 fallback）

记住这个结果，后续 Step 4/5 要用。

---

## Step 2: 收集用户偏好

向用户询问以下 **4 个问题**。

### 呈现方式（按 agent 分支）

#### 如果你是 Claude Code
- Cowork 环境有 AskUserQuestion 工具：用它一次性发出 4 个问题
- 普通 Claude Code：在聊天里输出 markdown 编号列表

#### 如果你是 Cursor / Codex
在聊天里输出 markdown 编号列表，等用户文字回复

#### 如果都不是
同上，markdown 编号列表

### 4 个问题（所有 agent 一致）

**问题 1: 使用场景（必填，单选）**

```
A. 工作记忆（业务决策、会议、商家、需求）
B. 个人项目（健身、读书、旅行、家庭）
C. 两者都要
```

→ 决定启用哪些 preset。

⚠️ 选「使用场景」而非「身份」的原因：身份不稳定（学生可能在公司实习、PM 可能下班搞副业），
场景才是真正决定记什么的因素。同一个人在工作和个人项目里要分别建独立 vault 也是合理的。

⚠️ MVP 阶段只有 `professional-work` 可用。如果用户选 B/C，告诉用户：
> "个人场景的 preset 还在开发，先用工作场景试试可以吗？"

未来会按用户角色推 page_type 组合（详见 `docs/future-ecosystem.md` 方向 1）。

**问题 2: 数据目录位置（必填，单选 + 自定义）**

呈现选项**前**，agent 应先 ls 用户家目录，发现已有 vault 就把对应选项排第一：

```
A. ~/business-memory/         （推荐，标准位置）
B. ~/Documents/business-memory/  （macOS 用户常见，方便 iCloud 同步）
C. <已有 Obsidian vault>/business-memory/   （如检测到 ~/Obsidian 或类似）
D. 自定义路径
```

**问题 3: 隐私模式（必填，单选）**

```
A. 本地（不推送任何 git remote，最安全）
B. 私有 git remote（公司外的私有仓库，安装后自行配 remote）
C. 公司内网 git remote（公司内部 GitLab/Gitea）
```

→ 决定 `.gitignore` 内容和 git remote 提示。

**问题 4: 现有数据迁移（必填，是/否）**

```
A. 是（请告诉我现有笔记目录路径，安装后我会作为 raw 引入）
B. 否
```

记下回答，分别引用为 `{scenario}` `{data_dir}` `{privacy_mode}` `{migration_source}`。

---

## Step 2.5: page_type 共创对话（可选）

完成 4 个问题后，在创建目录之前，做一次轻量对话，让用户对默认 preset 有点感觉，并发现是否需要立即定制。

**触发**：所有 agent 都执行（不分 tier）。**跳过条件**：用户在问题 1 选了"两者都要"且 MVP 只有一个 preset 时直接告诉差异，本步可省。

**对话脚本**：

> 当前你启用了 `professional-work` preset，里面有 2 个 page_type：
> - `decision`：业务决策（一个具体方案、选型、商业安排）
> - `meeting`：会议或对齐
>
> 你日常工作里，除了「决策」和「会议」，还有哪些类型的事是你经常想回头查的？比如：
> - 商家档案 / 客户档案
> - 需求 spec
> - 架构方案
> - 跟踪类（OKR / 项目 milestone）
> - 别的
>
> 如果有，告诉我大致是什么；如果没有想好也没关系，先用默认两个跑起来，后面随时可以加。

根据用户回答：

- **回答"先用默认"**：直接进 Step 3
- **回答"先用默认，但后续可能加 X"**：
  - 先按默认两个 type 安装
  - 把"X 可能要建"记录到内存，Step 3 后写入 `inbox/pending-review.md` 的"未来扩展"段
  - 不要立刻新建 yaml
- **回答"想加 X"**：
  - 不要立刻新建 yaml（避免空 type 污染）
  - 把"X 可能要建"写到 `inbox/pending-review.md` 的"未来扩展"段（先不创建 inbox 目录，记在内存中，Step 3 后写入）
  - 提示："等你 sink 第一条相关 raw 时，告诉我'这条应该是 X type'，我会按你的描述帮你新建 page_type yaml。"

**为什么不在 setup 阶段直接让用户写 yaml**：
- 用户没有真实 raw 在手时，写出来的 type 描述容易过度抽象 / 错位
- 真要建新 type，留到第一条该 type 的 raw 出现时由 agent 协助生成（用上下文约束生成质量）

---

## Step 3: 创建数据目录 + 配置文件

**通用步骤（所有 agent 一致）**：

```bash
mkdir -p {data_dir}/raw {data_dir}/decisions {data_dir}/meetings {data_dir}/inbox
cd {data_dir}
git init
```

`inbox/` 目录用于持久化待人工 review 的项（sink 流程产生的 REVIEW、Step 2.5 收集的"未来扩展"建议都写到 `inbox/pending-review.md`）。

写 `{data_dir}/config.yaml`：

```yaml
schema_version: 1
kit_path: {kit_clone_path}        # 本 wiki-mem-kit 的 clone 位置
presets:
  - professional-work             # MVP 只此一个
data_root: .

overrides:
  page_types: []
  skill_additions: []
  frontmatter_overrides: {}
```

按 `{privacy_mode}` 写 `.gitignore`：

#### A: 本地模式
```gitignore
# 本地模式仍然跟踪业务 markdown。
# 防误推靠不配置 git remote；不要用白名单忽略业务文件。
.DS_Store
*.swp
*.tmp
.idea/
__pycache__/
*.pyc
```
另外提示用户："你选了本地模式，不要 git remote add 任何 remote。"

#### B 或 C: git remote 模式
```gitignore
# 标准忽略
.DS_Store
*.swp
*.tmp
.idea/
__pycache__/
*.pyc
```
提示用户："数据目录的 git 仓库已建好，请用 `git remote add origin <你的私有仓库地址>` 配置 remote。"

如果 `{migration_source}` 不为空，把现有笔记目录复制到 `{data_dir}/raw/_imported/` 下，提示用户后续可以用 sink skill 处理。

如果 Step 2.5 收集到了"未来可能要加的 page_type / 扩展方向"，在 Step 3 末尾初始化或追加写入 `{data_dir}/inbox/pending-review.md`：

```markdown
# Pending Review

未确认的 sink / migrate / setup 项目。处理后请在该项前打 ✅ 或直接删行。

## 未来扩展（来自 setup Step 2.5）

- [ ] 可能需要新增 page_type: merchant-profile。等第一条相关 raw 出现时，再基于真实材料设计 yaml。
```

如果用户没有提出扩展方向，也建议初始化同一个文件并写：

```markdown
# Pending Review

未确认的 sink / migrate / setup 项目。处理后请在该项前打 ✅ 或直接删行。

## 未来扩展（来自 setup Step 2.5）

（用户暂未提出）
```

---

## Step 4: 加载 Skill（按 agent 分支）

请按 Step 1 识别出的 agent 类型执行对应分支。

### 如果你是 Claude Code

```bash
mkdir -p {data_dir}/.claude/skills
ln -s {kit_path}/presets/professional-work {data_dir}/.claude/skills/professional-work
```

验证：在 `{data_dir}` 下启动 Claude Code 应能看到 `professional-work` skill 可用。

### 如果你是 Cursor

```bash
mkdir -p {data_dir}/.cursor/rules
```

**1) 把 sink 主流程复制为入口规则**：把 `{kit_path}/presets/professional-work/SKILL.md` 内容复制到 `{data_dir}/.cursor/rules/wiki-mem-professional-work.mdc`，并在文件**最顶部**加 frontmatter：

```yaml
---
description: wiki-mem-kit professional-work preset for business memory sink
globs: ["raw/**/*.md", "**/*.md"]
alwaysApply: false
---
```

> globs 只列 `raw/**/*.md` 作为关键触发，再加 `**/*.md` 兜底所有 markdown。**不要**列死 `decisions/**` `meetings/**`——下面 yaml 是动态扫描的，硬编码会让 migrate 新增的 type folder（如 merchant-profiles/）从 Cursor 视野里消失。

**2) 动态附加所有 page_type yaml**：扫描 `{kit_path}/presets/professional-work/*.yaml`（**不要**列死 decision/meeting；migrate 新增的 yaml 也要包括），把每个 yaml 内容作为 markdown 段附在 `.mdc` 文件下方，用 ` ```yaml ` 围栏包起来，每段前加一行 `### page_type: <yaml.name>`。

**3) 重装时刷新**：用户跑 migrate 加新 type 后，建议重新执行本步（或在 migrate 完成时提示"请重跑 setup 同步 agent 入口"）——保持 `.cursor/rules/wiki-mem-professional-work.mdc` 与 kit 当前 yamls 一致。

### 如果你是 Codex

往 `{data_dir}/AGENTS.md` 追加：

```markdown
## wiki-mem professional-work preset

When user asks to sink raw materials from `raw/` folder, follow the sink procedure documented at:
`{kit_path}/presets/professional-work/SKILL.md`

Page types are defined dynamically. Scan `{kit_path}/presets/professional-work/*.yaml` to enumerate
all enabled types; each yaml's `name` field gives the type, and `folder` field (or default `<name>s`)
gives the output directory. Do NOT hardcode "decision" / "meeting" / "decisions/" / "meetings/" —
migrate may have added new types like merchant-profile, and they must be respected.

Output written to each type's resolved folder under `{data_dir}/`.
REVIEW block printed to chat for user confirmation.
```

如果 `{data_dir}/AGENTS.md` 不存在则新建。

> **重装提醒**：用户跑 migrate 加新 type 后，本段不需要改（因为是动态扫描），但建议运行 health check 确认 agent 实际能读到新 yaml。

### 如果都不是 (Tier 3 fallback)

告诉用户：

> "你的 agent 不在 wiki-mem-kit 自动加载列表（目前自动支持 Claude Code / Cursor / Codex）。
> 每次想 sink 时，请告诉它：'读取 `{kit_path}/presets/professional-work/SKILL.md` 并按其指令执行。'"

把这条提示写到 `{data_dir}/HOW-TO-USE.md` 顶部，防止用户忘记。

---

## Step 5: 配置 MCP server（按 agent 分支）

⚠️ **MVP 阶段 MCP server 还未实现**。本 Step 暂时跳过，告诉用户：

> "读取功能（MCP server）正在开发中。MVP 阶段，agent 想读取你的业务记忆时，请明确告诉它：
>
>   '读取 `{data_dir}/config.yaml` 找到 enabled presets，扫描 `{kit_path}/presets/<preset>/*.yaml` 拿到所有 page_type，按每个 yaml 的 folder 字段（缺省 `<name>s`）解析目录，从这些目录读取上下文。'
>
> 默认目录至少包括 `decisions/` 和 `meetings/`；migrate 新增的 type（如 `merchant-profile` 落到 `merchants/` 或 `merchant-profiles/`）也都要扫到。**不要硬编码两个目录**。"

未来 MCP server 实现后，本 Step 按以下分支执行：

### 如果你是 Claude Code
编辑 `{data_dir}/.mcp.json`（项目级）添加：
```json
{
  "mcpServers": {
    "wiki-mem": {
      "command": "wiki-mem-mcp",
      "args": ["--data-dir", "{data_dir}"]
    }
  }
}
```

### 如果你是 Cursor
编辑 `{data_dir}/.cursor/mcp.json` 添加同上结构。

### 如果你是 Codex
编辑 `~/.codex/config.toml` 追加：
```toml
[mcp_servers.wiki-mem]
command = "wiki-mem-mcp"
args = ["--data-dir", "{data_dir}"]
```

### 如果都不是
打印上述 JSON / TOML 片段，告诉用户："请按你 agent 的 MCP 配置方式手动添加这个 server。"

---

## Step 6: 健康检查 + 输出 next steps

执行检查清单：

- [ ] `{data_dir}/config.yaml` 存在且 YAML 可解析
- [ ] `{data_dir}/config.yaml` 包含 `schema_version: 1`、`kit_path`、`presets`、`data_root`
- [ ] `{data_dir}/raw/`、`decisions/`、`meetings/`、`inbox/` 四个目录存在
- [ ] `{data_dir}/.git/` 存在（git 已初始化）
- [ ] `git check-ignore inbox/pending-review.md` 退出码非 0（即文件未被 .gitignore 吞）
- [ ] `git ls-files --error-unmatch config.yaml .gitignore` 成功（说明已 `git add`），或 `git status --short` 显示为未跟踪等待提交。注意：`git status --short` 对**已跟踪且干净的文件**无输出，不能单独用它判断跟踪状态
- [ ] 按 agent 类型的 skill 加载文件存在（`.claude/skills/...` 或 `.cursor/rules/...` 或 `AGENTS.md`）

任何检查失败：清晰报告哪一步失败，告诉用户如何手动修复，**不要试图自动修复**（避免越权）。

成功后输出给用户：

```
✅ wiki-mem-kit 安装完成！

数据目录：{data_dir}
当前预设：professional-work
当前 agent：{agent_type}

下一步：
1. 把第一条原始材料整理成 markdown 放到 {data_dir}/raw/，例如：
   raw/2026-04-25-客户对齐.md

2. 然后告诉我："sink 一下 raw/2026-04-25-客户对齐.md"
   我会按 professional-work skill 把它沉淀成 decision 和 meeting page，
   并列出需要你确认的 REVIEW 项。

3. sink 完成后请定期检查 inbox/pending-review.md，里面会累计需要你确认的金额、承诺、冲突和未来扩展项。

4. 处理 inbox/pending-review.md 时，告诉我："跑一下 review"，我会按 `{kit_path}/review/SKILL.md` 一项一项过完所有未确认事实，把对应 page 从 needs_review 还原为 clean。**这是 needs_review → clean 的唯一标准入口**。

5. 业务记忆攒到一定量后（10 个 page 左右），可以让我帮你看看图谱是否有新发现。

6. 如需体检数据仓库，可告诉我："按 {kit_path}/health/SKILL.md 跑一次 health check"。

如有问题，请查看 {kit_path}/README.md。
```

---

## 异常处理

- **数据目录已存在且非空**：不要覆盖。问用户："检测到 {data_dir} 已存在数据，要修复 / 升级 / 还是换个目录重装？"
- **kit clone 失败**：报告网络问题，建议用户手动 clone 后再触发本 skill 并提供 `{kit_path}` 参数
- **git init 失败**：跳过 git 初始化，不阻塞其他步骤
- **symlink 失败（Windows 等）**：fallback 到 copy，并告诉用户 kit 升级时需手动同步

## 幂等性

如果用户重复触发本 skill：

1. 先检查 `{data_dir}/config.yaml` 是否存在
2. 存在则进入「已安装」分支：
   - 读出现有配置
   - 询问用户：修复 / 升级 preset / 换数据目录重装
3. 不存在则按正常流程从 Step 1 开始

## 参考

- 三层分层定义：`docs/agent-compat.md`
- page_type 规范：`docs/page-type-spec.md`
- 类型演化：`docs/type-evolution.md`
- 未来生态：`docs/future-ecosystem.md`
- sink 流程：`presets/professional-work/SKILL.md`
- review 闭环：`review/SKILL.md`（needs_review → clean 唯一入口）
- 健康检查：`health/SKILL.md`
- 迁移流程：`migrate/SKILL.md`
