# professional-work 预设

> **前置假设**：本 skill 由 agent 在用户已通过 `setup/SKILL.md` 完成安装后使用。
> 如果还没安装，请先执行 setup skill。
>
> **agent 兼容性**：本 sink 流程主要靠文件读写，对 agent 通用。如未来引入 agent 特化步骤，
> 必须遵循 `docs/agent-compat.md` 的三层分支语法。

适用场景：知识工作者的日常业务记忆，特别是需要在产品/技术/运营/商家之间穿梭的角色（如 AI 全栈、PM、TPM）。

包含两个 page_type：
- `decision.yaml` — 业务决策
- `meeting.yaml` — 会议与对齐

## 通用 sink 流程

每次有新 raw 进入 `raw/` 目录时，按以下步骤处理：

### Step 1: 阅读 raw + 已有 page 索引

读取新 raw 内容。然后**动态发现 type**：

1. 读 `{data_dir}/config.yaml`，拿到 `kit_path` 和 enabled presets 列表
2. 扫描 `{kit_path}/presets/<preset>/*.yaml`，对每个 yaml 解析 folder（按 page-type-spec 的统一约定 `yaml.folder ?? f"{name}s"`）。MVP 阶段 `config.yaml` 的 `overrides.page_types` 强制为空数组，无需合并；future 启用后按 `[{path: ...}]` 格式合并
3. 对每个 type folder，读取 `{data_dir}/<folder>/*.md` 的 title + aliases（不需要全文），作为本次 sink 的"已有 page 索引"

**不要**硬编码 `decisions/` 和 `meetings/`——动态发现保证 merchant-profile 等新 type 在 sink 时也能被读到、被匹配到，而不是被强行塞回 decision/meeting。

### Step 2: 判断涉及哪些 page_type

根据每个 page_type 的 `description` 中「何时归到这里」一段判断。
一条 raw 可能同时涉及多个 type（例：一个会议同时产生了 1 个 meeting page + 2 个 decision page + 1 个 merchant-profile page）。

判断时**遍历 Step 1 发现的所有 type**，不只 decision/meeting。新增的 type（如 merchant-profile）也参与判断。

### Step 3: 判断新建 vs 更新

**通用框架**：对每个涉及的 type，读取该 type yaml 中 description 的「更新规则」段，按其执行。

下面的两个 type 是 professional-work preset 自带的，举例说明（其他 type 类比，按各自 yaml 的更新规则）：

- **decision**（yaml 规则）：用 title/aliases 模糊匹配已有 page，匹配则更新
- **meeting**（yaml 规则）：永远新建

如果新 type（如 merchant-profile）的 yaml 更新规则是"按 merchant_name 匹配已有 page，匹配则更新；新增字段附加，不覆盖"，sink 必须按这条规则执行，**不要**用 decision/meeting 的逻辑套用。

> **架构纪律**：本 SKILL 的 sink 流程是**通用框架 + 已知 type 范例**。新增 type 时只改 yaml，不需要改本 SKILL.md（除非需要类型间交互，如 Step 3c 反向闭环）。

**decision 的更新必须先区分两类情况**（这是 decision-specific 的精细化，其他 type 不一定有这个区分）：
- **内部演化**：同一个方案的范围、约束、时间、商业模式等细节变化。更新原 decision，不新建。
- **外部取代**：旧方案作废、路线根本转向、或出现另一个可独立命名的新方案接替。新建 replacement decision，并把旧 decision 标记为 `status: superseded`。

外部取代时必须维护双向字段：
- 新 page 加 `supersedes: [旧 decision title]`
- 旧 page 加 `superseded_by: [新 decision title]`
- 旧 page 的 `status` 改为 `superseded`
- 新 page 的 `sources` 默认只放本次 raw；不要复制旧 page 的 `sources`，历史继承靠 `supersedes` 图边表达

#### Step 3a: 硬性相似度检查（避免重复 page）

在创建任何新 decision 前，必须做一次相似度检查：

1. 把待写的 `title` 和现有所有 decision 的 `title + aliases` 做字面相似度检查
2. 命中任一条件即视为疑似重复：
   - 待写 title 精确命中已有 title 或 alias
   - 待写 title 与已有 title 去掉空格、标点后完全相同
   - 两者共享连续 4 个及以上中文字符（例：`硬件接入方案`）
   - 两者的核心名词集合至少重合 2 个（例：`棋牌室`、`硬件`、`接入`）
3. 命中疑似重复时：
   - **不要**直接新建
   - 把这一对放进 REVIEW，问用户："这两个看起来像是同一个东西，要合并到 X 还是确认是新方案？"
   - 用户确认前，本次只把内容写到 `inbox/draft-{title}.md`，不入 `decisions/`

这条规则比"语义判断"硬，目的是防止模型在不同会议中给同一方案起略不同的 title 而产生孤儿 page。

#### Step 3b: 拆分原则

按各 page_type description 中「拆分原则」段执行。常见错误：
- 把"v1 范围 + 商业模式 + 时间线"塞进 3 个 decision page → 应是 1 个
- 把"硬件方案 + 计费方案"塞进 1 个 page → 应是 2 个

#### Step 3c: 外部取代时的反向闭环

当本次 sink 触发 Step 3 的「外部取代」分支（旧 decision 改 status: superseded）时，必须做一次反向扫描，让历史 meeting 上的 follow_ups 不再被读取侧误当成有效待办。

**触发条件**：本次 sink 中至少有一个 decision 被打 `status: superseded`。如果只是内部演化（同一 page 范围细节变化），不触发本步。

**步骤**：

1. 收集旧 decision 的 `sources` 数组（这些是历史上"产出该 decision"的 raw 文件）
2. 在 `meetings/` 下扫描所有 page，找出满足以下任一条件的 meeting：
   - meeting 的 `sources` 与旧 decision 的 `sources` 有交集
   - meeting 正文 wikilink 到旧 decision（按 title 或 alias 匹配）
3. 对每个命中的 meeting page：
   - **不要**直接删除或重写 `follow_ups` 数组（保留历史 frontmatter 的快照价值）
   - 在 page 末尾追加（或新增）`## 待确认事实` 段，写一行：「follow_ups 中的 X、Y、Z 因 [[新 decision title]] 取代 [[旧 decision title]] 可能已失效，需要确认是否标 obsolete」
   - 把 frontmatter 的 `review_status` 改为 `needs_review`（如果原本是 clean）
   - 在「演化时间线」末尾追加一行：`- {today} 因 [[旧 decision title]] 被取代，follow_ups 可能失效 → [[新 decision title]]`
4. 对每个被反向更新的 meeting，在本次 sink 的 REVIEW block 中加一条**带稳定前缀**的项：
   - `[follow_ups_obsolete] meetings/<file>.md: follow_ups 因 [[新 decision]] 取代 [[旧 decision]] 可能失效，请逐条确认 obsolete/keep`

   `[follow_ups_obsolete]` 前缀是 review/SKILL.md 路由依赖的稳定标记——**不要改写**这个前缀的拼写或位置（必须紧跟 `- [ ] ` 之后，独立方括号包裹）。

**为什么不自动改 follow_ups**：判断"哪几条 follow_up 真的失效"需要语义理解（有些 follow_up 可能跨方案仍然有效，比如"双方签订合作意向"在新 SaaS 方案里可能仍然成立）。Step 3c 只负责"提醒"，由用户在确认 REVIEW 时做最终判断。

**没有命中**：如果没有 meeting 引用旧 decision，跳过此步，但仍然需要在 REVIEW 中记录"已扫描，无需反向更新"。

### Step 4: 生成 page 内容

每个 page 必须包含：
- 完整的 frontmatter（必填字段不能漏）
- `schema_version: 1`
- `review_status: clean` 或 `review_status: needs_review`
- 至少一个 H2 段作为 compiled 区
- `## 演化时间线` 段，至少一行 `- {date} {摘要} → [[raw/{文件名}]]`

如果 page 的 `status: superseded`，正文 frontmatter 后必须先写 superseded block，再写原 compiled 区：

```markdown
> ⚠️ **本方案已 superseded**（YYYY-MM-DD）
> 本方案被「[[新方案 title]]」取代。原因：一句话说明路线为何变化。
> 下文保留为历史背景，**不再作为当前执行约束**。
```

不要删除旧 page 的「当前结论」和历史段落；它们是当时决策的快照。只在顶部 block 和「演化时间线」里标明已被取代。

文件名规则（按 type yaml description 中「更新规则」或「文件名格式」段执行；下面是已知 type 的范例）：
- decision：从 title 派生，kebab-zh（保留中文，空格转 -）
- meeting：`{meeting_date}-{对方主名}.md`
- principle：从 title 派生，kebab-zh，无前缀
- competitor-profile：从 competitor_name 派生，kebab-zh
- 新 type：默认从 page 主标识字段派生 kebab-zh（如 merchant-profile 用 `merchant_name`），具体由该 yaml 在 description 中声明

#### Step 4b: follow_up 写法约定

`follow_ups` 数组每一项是字符串，但**有内部规范**：

- **明确行动**（执行前不需要再决策）：直接写一句话，例：
  - `给王五硬件清单（5/2 前）`
- **行动 + 未解决前置问题**：在括号里标注"需先确认"，让读取侧（包括未来的你）一眼看出这条不能直接干：
  - `准备 1-2 个 demo 用于下次见中田（**需先确认**：demo 场景是正式课记录还是体验课展示？由谁主导？）`
  - `推进美团内部立项（**需先确认**：商业模式是 SaaS 订阅还是技术服务？）`
- **由他人执行的待跟进**：明确动作主语：
  - `王五月底前确认嘉宝是否开工`

不规范的反例：
- ❌ `准备 demo`（缺时间、缺范围、内部还有未决问题）
- ❌ `推进立项`（推进什么阶段不清楚）
- ❌ `跟进商家`（缺谁、缺动作）

**为什么强制标注前置问题**：follow_ups 字段是机器可读的"等待执行"列表（未来 MCP/agent 会消费它）。如果一条 follow_up 表面上是 active 但实际上需要先做决策才能执行，agent 把它当 todo 直接催办会很尴尬。括号注明"需先确认"让该项语义透明：**它是"需要先决策再行动"的混合项，不是纯执行项**。

如果一个 follow_up 的前置问题非常重要、影响多个人，应该把它**升级**到 ## 待确认事实 段，并写入 inbox 让 review skill 闭环。括号注明只用于"轻量级的内部子问题"。

### Step 4a: REVIEW 事实写入规则

REVIEW 不阻塞写入，但会改变 page 的语义：

- 未命中金额、对外承诺、冲突、身份不确定、方案取代等 REVIEW：`review_status: clean`
- 命中高风险 REVIEW：`review_status: needs_review`
- 高风险事实不要用确定语气写进「当前结论」或「会议要点」
- 高风险事实必须写入 `## 待确认事实` 段，并同步追加到 `inbox/pending-review.md`

高风险事实包括：
- 金额、报价、预算、薪资、KPI 数字
- 对外承诺、交付时间、合作协议、排期承诺
- 与已有 page 的当前结论冲突
- 人名 ID 不确定且可能影响后续关系图谱
- decision 被 superseded 后旧方案的承诺、金额、待办是否需要主动通知

用户之后确认某条 REVIEW 时，才能把对应事实从 `## 待确认事实` 移入「当前结论」或「会议要点」，并在没有其他待确认事实时把 `review_status` 改回 `clean`。

### Step 5: 输出 REVIEW + 持久化到 inbox/pending-review.md

在所有 page 写入完成后：

**1. 持久化 REVIEW**：把本次产生的 REVIEW 项追加（append，不要覆盖）到 `inbox/pending-review.md`。

文件格式（首次创建时写表头）：

```markdown
# Pending Review

未确认的 sink 项目。处理后请在该项前打 ✅ 或直接删行。

## 2026-04-25 (来自 sink raw/2026-04-25-王五会议.md)

- [ ] decisions/嘉宝棋牌室硬件接入方案.md: 「3 万预算」是否对外承诺给王五？需要确认。
- [ ] meetings/2026-04-25-王五.md: mentions_people 出现张三（朝阳大悦城无人棋牌室），是否需要建独立 merchant page？
```

**2. 在聊天中输出同样的 REVIEW block**，让用户当场看到：

```
REVIEW（已追加到 inbox/pending-review.md）:
- decisions/嘉宝棋牌室硬件接入方案.md: 「3 万预算」是否对外承诺给王五？
- meetings/2026-04-25-王五.md: 张三是否需要建独立档案？
```

REVIEW 不阻塞写入，但高风险 REVIEW 会让相关 page 标记为 `review_status: needs_review`，并把对应事实写入 `## 待确认事实`，避免读取侧把它误当硬约束。
持久化的目的：用户隔天回来还能找到没确认的项，不会因为聊天上下文丢失而漏。

### Step 6: page_type 涌现评估（隐式后台步骤）

每次 sink 末尾跑一遍——评估"本次 raw 是否催生新 page_type 候选"。**判断标准、阈值、置信度分级全部按 `docs/type-emergence.md` 执行**，本步只描述具体动作。

#### 6.1 三个触发信号检查

按 `type-emergence.md` 的 Step A 检查：

- **塞不下信号**：本次 sink 是否把某个有命名的具体实体强行塞进现有 type 的 body 段或 REVIEW？
- **复发信号**：同类实体在过去 raw 或 `inbox/pending-review.md` 的「page_type 候选」段出现过 ≥1 次？
- **显式信号**：用户在 raw 中是否说"以后要追踪/反复发生/单独建档/长期跟"等话？

按 Step B 阈值组合判断是否进入候选评估，命中 0 个或仅 1 个非显式信号则跳过本步全部后续。

#### 6.2 演化性 gate（强制，命中即否决）

按 `type-emergence.md` Step B.5：判断候选实体未来是否有独立演化路径。

- **持续追踪对象**（多次接触意图、关系会演化、信息会增量补充）→ 通过，进入 6.3
- **一次性事件**（仅此次出现、无后续接触锚点、信息已给完）→ **否决**，写 `[否决] {名}: 演化性预判为单次事件`，不进 6.3
- **不确定** → 保留但置信度强制降到"弱"，标"演化性待观察"

启发式：用户是否提到"以后/下次/继续/持续"？raw 中是否有联系方式/合作意向/未决议题等延续锚点？是否有专门的人格化属性（名字/角色/资源）？

#### 6.3 区分度检查

按 `type-emergence.md` Step C：列出候选预期字段集合，与现有每个 type 字段两两比较：

- 重合 ≥ 70% → **改写为"给现有 X type 加字段 Y"提案**（type-evolution 场景 2）
- 重合 ≤ 30% → 保留为新 type 候选
- 中间档 → 置信度降一档，附"边界类，先攒数据"标签

#### 6.4 体量预估

按 `type-emergence.md` Step D 估算 6 个月内 page 数：

- < 5 → **否决**：不写候选，但留一行 `[否决]` 注记，避免后续 sink 反复评估同一概念
- 5-20 → 写候选，标"draft 模式"，建议先攒到 `inbox/draft-{type}-N.md`
- ≥ 20 → 写候选，标"立即正式化"

#### 6.5 抑制重复

写入 `inbox/pending-review.md` 前，先扫描「page_type 候选」段：

- 已存在同名候选 → **不追加新行**，但**更新已有行**（递增触发次数、追加最新 raw 引用、按需提升置信度）
- 已存在 `[否决]` 同名注记 → 跳过本次评估（除非新信号强度显著高于之前）

#### 6.6 写入格式

附加（append）到 `inbox/pending-review.md` 的「page_type 候选（来自 sink 自动评估）」段。如该段不存在，创建之。每条候选格式：

```markdown
- [ ] [置信度: 强/中/弱] 提议新建 `<type-name>` （或 给现有 `<type>` 加字段 `<field>`）
  - 触发信号：<塞不下/复发/显式 的具体描述>
  - 区分度：与现有 X type 重合 N%、与 Y type 重合 M%
  - 体量预估：6 个月内约 K 个 page
  - 建议字段（草案）：<field-list，仅新建 type 时填>
  - 当前 raw 引用：<raw-file>
  - 处理建议：<立即正式化 / draft 攒数据 / 改加字段 / 边界类待复发>
```

#### 6.7 在聊天中提示

sink 完成时，如本次产生了新候选，在最后的 REVIEW block 之后追加一段：

```
TYPE PROPOSAL（已写入 inbox/pending-review.md）:
- [置信度: 强] merchant-profile（新建）
- [置信度: 弱] decision 加 commercial_model 字段
```

如果没有新候选，**不输出**这段，避免噪音。

#### 6.8 失败兜底

如果模型对某个候选无法判断置信度或字段重合度，**降一档置信度**输出，宁可让用户多 review 一次也不要让候选静默丢失。

## REVIEW 通用规则

无论哪个 page_type，命中以下任一条件必须列入 REVIEW：

- **金额数字** — 任何涉及钱的数字（预算、报价、KPI 数字、薪资）
- **对外承诺** — 向商家、其他团队、客户做的承诺或时间表
- **与现有结论矛盾** — 新事实与已有 page 的「当前结论」冲突
- **人名 ID 不确定** — 出现新人名但无法确认是哪位（重名风险）
- **方案被取代** — decision 改为 `superseded` 时，列出旧方案里的对外承诺、金额和待办，问用户是否需要通知相关方或更新计划

命中金额、对外承诺、冲突、方案被取代这几类时，必须把 page 标记为 `review_status: needs_review`。

各 page_type 在自己的 `description` 中可加额外的 REVIEW 触发条件。

## 文件结构约定

详见 `wiki-mem-kit/docs/page-type-spec.md` 的「通用文件结构」段。

要点：
- frontmatter（YAML） + 多个 H2 段 + 必有的 `## 演化时间线` 段
- 时间线行格式：`- YYYY-MM-DD 一句摘要 → [[raw/原始文件]]`
- 摘要必须是人能扫一眼读懂的，不要只放链接
- `sources` 只放该 page 的直接 raw 证据；被旧方案影响、继承旧方法论或取代旧方案时，用 wikilink / `supersedes` 表达关系，不复制旧 raw 到新 page
- `## 待确认事实` 只在 `review_status: needs_review` 时出现；不要把待确认事实混在确定结论里

## 完整写入示例

**输入** — `raw/2026-04-25-王五会议.md`：

```
今天和王五对齐了棋牌室硬件状态板的事，决定先做最简版只接灯光和门禁，
计费系统留 v2。预算 3 万。王五说他下周给硬件清单。
```

**应该产生的 sink 输出**：

1. **新建** `decisions/棋牌室硬件状态接入方案.md`
   ```markdown
   ---
   title: 棋牌室硬件状态接入方案
   type: decision
   schema_version: 1
   sources: [raw/2026-04-25-王五会议.md]
   status: active
   review_status: needs_review
   decision_date: 2026-04-25
   ---

   ## 当前结论

   v1 范围：仅接入灯光控制 + 门禁状态
   v2 留空：计费系统暂不集成

   ## 待确认事实

   - 预算 3 万是否是对外承诺，需要确认。

   ## 演化时间线

   - 2026-04-25 王五同意先做硬件状态板 v1 范围 → [[raw/2026-04-25-王五会议]]
   ```

2. **新建** `meetings/2026-04-25-王五.md`
   ```markdown
   ---
   title: 2026-04-25 王五对齐
   type: meeting
   schema_version: 1
   sources: [raw/2026-04-25-王五会议.md]
   review_status: needs_review
   attendees: [王五, 我]
   meeting_date: 2026-04-25
   follow_ups:
     - 王五下周给硬件清单
   ---

   ## 会议要点

   对齐了棋牌室硬件状态板的方案。决议见 [[决策-棋牌室硬件状态接入方案]]。

   ## 待确认事实

   - follow_up「王五下周给硬件清单」是否需要进入提醒系统。

   ## 演化时间线

   - 2026-04-25 会议召开 → [[raw/2026-04-25-王五会议]]
   ```

3. **REVIEW block**
   ```
   REVIEW:
   - decisions/棋牌室硬件状态接入方案.md: 「3 万预算」是否对外承诺给王五？需要确认。
   - meetings/2026-04-25-王五.md: follow_up「王五下周给硬件清单」需要确认是否要排进我的提醒。
   ```

## 添加新 page_type 的流程

1. 在 `presets/professional-work/` 下加 `<new-type>.yaml`
2. 写 description 时确保覆盖五段（是什么/必填字段/何时归类/更新规则/REVIEW 触发）
3. 控制 description 在 100 字内
4. 不需要改本 SKILL.md（除非新 type 需要破例的特殊流程）
