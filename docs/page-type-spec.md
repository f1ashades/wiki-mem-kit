# page_type 定义规范

page_type 是 wiki-mem-kit 唯一的扩展点。每个 page_type = 一个 YAML 文件，放在 `presets/<preset_name>/` 目录下。

## vault config v1

每个数据仓库根目录必须有 `config.yaml`。MVP v1 固定格式：

```yaml
schema_version: 1
kit_path: ~/.local/share/wiki-mem-kit
presets:
  - professional-work
data_root: .

overrides:
  page_types: []
  skill_additions: []
  frontmatter_overrides: {}
```

字段含义：
- `schema_version`：配置文件 schema 版本，当前固定为 `1`
- `kit_path`：wiki-mem-kit 的本地路径，可用绝对路径、相对路径或 `~`，不要使用旧字段 `kit`
- `presets`：启用的 preset 列表
- `data_root`：raw/page 相对配置文件的位置，MVP 固定推荐 `.`
- `overrides`：私有覆盖入口，MVP 可为空但字段必须存在

### overrides 子字段 MVP 契约

```yaml
overrides:
  page_types: []          # MVP: 必须为空数组
  skill_additions: []     # MVP: 必须为空数组
  frontmatter_overrides: {}  # MVP: 必须为空对象
```

**`page_types`（vault 私有 type 注册表）**：

- **MVP 阶段强制 `[]`**——sink/migrate/health/review 各处提到"合并 overrides.page_types"是为未来留接口，当前不应有任何元素。健康检查发现非空数组会输出 P1 警告"未来字段被提前使用"
- **future 格式**（v0.2 实现）：每项是一个对象 `{path: <yaml 文件相对 data_dir 的路径>}`，例：
  ```yaml
  page_types:
    - path: page-types/internal-process.yaml
  ```
  vault 私有 yaml 必须按 page-type-spec 的 yaml schema 编写。kit/preset 同名 type 优先于 vault override（防止用户意外覆盖 kit 上游规则）

**`skill_additions`（vault 自定义 skill 入口）**：

- **MVP 阶段强制 `[]`**
- **future 格式**：每项 `{path: <skill 路径>, description: <一句描述>}`，agent 按 setup 类似流程注册

**`frontmatter_overrides`（page_type 字段语义微调）**：

- **MVP 阶段强制 `{}`**
- **future 格式**：`{<type-name>: {<field>: {required: bool, default: ...}}}`，允许 vault 把 kit 默认可选字段在本仓库变成必填，或反之

**为什么 MVP 强制空**：早期把私有覆盖加进来会让 sink/migrate 行为依赖 vault 状态（"为什么我的 vault 行为和别人不一样"），调试困难。先让 kit preset 路径单一稳定，等真实 vault 私有需求积累到 3+ 个再设计可解析的格式。

## 字段

### name (必填)

字符串。该 type 的稳定标识符，用于 MCP 工具过滤、文件路径派生、跨 page 引用。

约定：小写英文，多词用连字符。例：`decision`、`merchant-profile`。

**为什么必须 YAML**：MCP 工具如 `recall(type='decision')` 要靠它做参数过滤，模型每次起的名字可能不一致。

### schema_version (必填)

整数。该 page_type 定义自身的版本号，当前 MVP 从 `1` 开始。

当 page_type 新增必填字段、改变字段语义、或改变写入规则时，必须升级 `schema_version`，并由 `migrate/SKILL.md` 指导旧 page 回填。新增可选字段不需要升级。

### description (必填)

多行字符串。给模型看的自然语言说明，应包含以下五段（顺序无所谓但都要有）：

- **是什么** — 一句话定义，举一个具体例子
- **必须的 frontmatter** — 列出必填字段和可选字段
- **何时归到这里** — 模型据此判断 raw 是否应写入此 type
- **更新规则** — 新建还是更新已有？怎么合并？
- **REVIEW 触发** — 哪些情况强制人工确认

如果该 type 可能被错拆或错合，再加一段：

- **拆分原则** — 一个 page 的颗粒度是什么；什么情况要拆，什么情况要并

**为什么必须 YAML**：模型理解新 type 的唯一锚点。

**写作纪律**：description 控制在 200 字内（不含示例）。超出说明该拆。

## 可选字段

### folder (可选)

字符串。该 type 的 page 默认存放目录。

**默认值规则**：`folder = name + s`（如 `decision` → `decisions/`，`merchant-profile` → `merchant-profiles/`）。

仅在默认推导不合理时显式覆盖（例：`merchant-profile` 想放到 `merchants/`）。

**统一解析约定**（kit 内 sink / migrate / health / review 都遵守）：

```
resolve_folder(yaml):
  return yaml.folder if yaml has folder field, else f"{yaml.name}s"
```

任何工具读 yaml 后引用该 type 的目录、git add 路径、扫描目标，**必须经此解析**，不能直接用 `<name>s`。覆盖与默认两种情况下都得到正确路径，且各组件保持一致。

## 通用文件结构

所有 type 的 page 都遵循同一个壳，无需在 page_type 里重复定义：

```markdown
---
title: 嘉宝棋牌室硬件接入方案
type: decision
schema_version: 1
sources:
  - raw/2026-04-25-王五会议.md
status: active
review_status: needs_review
aliases:
  - 王五硬件状态板项目
mentions_people:
  - 张三
---

## 当前结论

v1 范围：仅接入灯光控制 + 门禁状态
v2 留空：计费系统暂不集成

## 待确认事实

- 预算 3 万是否是对外承诺，待用户确认。

## 演化时间线

- 2026-04-25 与王五对齐 v1 范围 → [[raw/2026-04-25-王五会议]]
```

- 顶部 frontmatter 之后的所有 H2 段为 compiled 区
- `## 演化时间线` 为 timeline 区，每行一条事件，格式：`- YYYY-MM-DD 摘要 → [[raw/原始文件]]`

## frontmatter 通用字段（所有 page 共有）

这些字段在 kit 层定义，所有 page_type 自动继承：

- `title` — 必填
- `type` — 必填，等于 page 所属 page_type 的 name；默认从 folder 推导
- `sources` — 必填，数组，引用的 raw 文件路径列表
- `schema_version` — 必填，整数。该 page 写入时遵循的 schema 版本号；未来 type 变更时迁移工具据此判断是否需升级
- `review_status` — 必填，`clean` 或 `needs_review`。只要 page 中存在未确认的金额、承诺、冲突或身份判断，就必须是 `needs_review`

各 page_type 在 description 中可声明额外字段（如 decision 的 `status`、meeting 的 `attendees`）。

## 通用 frontmatter 约定字段（强烈推荐）

不强制，但跨 type 通用，先在此规范：

### aliases (数组)

该 page 的别名列表，用于 sink 阶段的「title 模糊匹配」和读取阶段的 wikilink 重定向。

定位：**「换一个名字指同一个东西」**——典型来源：
- 用户口头叫法 vs 正式 title（"王五硬件项目" → "嘉宝棋牌室硬件接入方案"）
- 历史 title（被改名前的）
- 同一对象的简称、缩写、英文名

加 aliases 后，sink 时模型从 raw 中识别出"王五硬件项目"会自动找到正式 page，不会新建一个。

### tags (数组)

分类标签，用于读取阶段的批量过滤（"所有跟棋牌室相关的 decision"）。

定位：**「同一类东西的标记」**——典型用法：
- 业务领域（`棋牌室`、`商家结算`、`物流`）
- 项目阶段（`样板店`、`已完结`、`v2 待启动`）
- 协作对象类型（`商家对齐`、`内部架构`）

### tags vs aliases 对比

| 维度 | aliases | tags |
|------|---------|------|
| 解决的问题 | 重名 / 别称 | 分类 / 检索 |
| 基数 | 通常 1-3 个 | 通常 2-5 个 |
| 唯一性 | 同一 vault 内不应有别的 page 用同名 alias | 同一 tag 期望被多 page 共用 |
| sink 时的作用 | 用于模糊匹配防重 | 不参与匹配 |
| 读取时的作用 | wikilink 重定向 | 过滤聚合 |

混用会导致：图谱噪声变大，搜索结果不准。

### mentions_people (数组)

会议或决策中**提到但不主导**的人，用于人物图谱构建。

不要与 `attendees`（meeting 的实际参会者）混淆。

举例：会议中王五提到他朋友圈有个张三在朝阳大悦城开棋牌室
→ `attendees: [王五, 我]`，`mentions_people: [张三]`
张三未来真要建独立档案时，我们能从这个字段反查"张三第一次出现在哪条 raw"。

### supersedes / superseded_by (数组)

用于表达 page 之间的取代关系，尤其是业务方案、决策、架构路线发生根本转向时。

- `supersedes`：当前 page 取代了哪些旧 page。写在新 page 上。
- `superseded_by`：当前 page 被哪些新 page 取代。写在旧 page 上。

典型用法：

```yaml
# 新 decision
supersedes:
  - 嘉宝棋牌室硬件接入方案

# 旧 decision
status: superseded
superseded_by:
  - 多店统一 SaaS 平台
```

写入纪律：
- `sources` 只记录该 page 的直接 raw 证据，不要为了继承历史而复制旧 page 的 sources
- 历史继承关系通过 `supersedes` / `superseded_by` 图边表达
- 被取代的 page 不删除，保留旧结论和时间线，便于回看当时为什么这么定
- 当 `status: superseded` 时，page 正文顶部必须有醒目的 superseded block，说明被谁取代、何时取代、旧结论是否仍可作为历史背景

### review_status 与待确认事实

`review_status` 是读取侧判断 page 是否可作为硬约束的最低信号。

- `clean`：当前 page 没有未处理 REVIEW 项；读取侧可以把「当前结论」当作普通业务上下文
- `needs_review`：当前 page 存在未确认的金额、对外承诺、与旧结论冲突、人名身份不确定、或方案取代后的次生影响；读取侧不能把这些信息当硬约束

写入纪律：
- 命中 REVIEW 的事实不要直接混进「当前结论」的确定语气
- 高风险但仍有记录价值的信息放到 `## 待确认事实` 段，并同步写入 `inbox/pending-review.md`
- 用户确认后，才能把对应内容移入「当前结论」，并把 `review_status` 改回 `clean`
- 如果只有少量低风险 REVIEW（例如是否需要建独立档案），可保留 `clean`，但金额、对外承诺、冲突必须标 `needs_review`
- 用户处理 REVIEW 走 `review/SKILL.md` 交互流程；这是 needs_review → clean 的唯一标准入口

**读取侧契约**（agent / MCP 消费 page 时必须遵守）：

- `review_status: clean` → frontmatter 各字段（含 `follow_ups`、`status` 等）可信，读取侧可直接消费
- `review_status: needs_review` → frontmatter **不权威**，读取侧必须先扫 `## 待确认事实` 段
  - 例：meeting page 在 superseded 反向闭环后会标 needs_review，此时 `follow_ups` 数组里可能包含已失效但还未被 review 处理的项
  - 正确读法：先看 `## 待确认事实` 是否提到 follow_ups 失效，命中则把对应项当"待用户确认"而不是活跃 TODO
- `obsolete_follow_ups`（如果存在）→ 永远当历史，不当活跃 TODO；该字段由 `review/SKILL.md` 在用户确认 follow_up 失效后从 `follow_ups` 移入

这条契约的目的：让未来 MCP / agent 在解析 frontmatter 时不会把"已经被反向闭环标记但还没被 review 处理"的 follow_ups 误当成活跃待办。

## wikilink 解析规则

`[[xxx]]` 是 page 之间引用的核心机制。kit 层的解析约定：

### 解析顺序

按以下顺序找目标 page：

1. **精确文件名**（最稳）：`[[嘉宝棋牌室硬件接入方案]]` 找 `decisions/嘉宝棋牌室硬件接入方案.md`
2. **精确 title**：`[[嘉宝棋牌室硬件接入方案]]` 匹配某 page 的 frontmatter `title` 字段
3. **精确 alias**：`[[王五硬件项目]]` 匹配某 page 的 aliases 数组
4. **fallback：sources 反查**：找不到目标时，看是否在 raw 引用同一份 source 的 page 中，给一个最佳匹配

匹配到多个时：报警告，列出候选，不自动选。

### 写 wikilink 的纪律

- **首选精确文件名**（最稳，移植时不破）
- 引用 raw 用 `[[raw/2026-04-25-王五会议]]`（带子目录前缀，区分 raw 和 page）
- 不要写自然语言短语 `[[这个方案]]`，这种解析必失败

## schema_version 与迁移

`schema_version` 是 page 写入时遵循的 page_type schema 版本号。

- 新建 page 时填当前 kit 中该 type 的版本号
- preset 升级、新增必填字段时，`migrate/SKILL.md` 会扫描所有 schema_version 落后的 page 并执行迁移
- 迁移完成后更新 page 的 schema_version

详见 `docs/type-evolution.md`。
