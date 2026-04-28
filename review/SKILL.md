# wiki-mem-kit Review Skill

> **触发条件**：用户对 agent 说类似：
> - "处理一下 pending review"
> - "把 inbox 里的 review 项跑一遍"
> - "我想 review 上次 sink 的待确认事项"
> - "把 needs_review 的 page 整理一下"
>
> **目标**：把 `inbox/pending-review.md` 里堆积的 REVIEW 项逐条交给用户决定，并把对应 page 的 `## 待确认事实` 段处理掉，把 `review_status` 还原回 `clean`。
>
> **agent 兼容性**：纯文件读写，对所有 agent 通用。

---

## 设计原则

- **闭环唯一入口**：`needs_review → clean` 必须走本 SKILL，禁止 sink / migrate 流程顺便改 review_status
- **逐项交互**：默认一项一项过，每项让用户做选择；用户可以一次说"全跳过到下一个 page"做批量动作
- **不替用户判断**：本 SKILL 不评估事实真伪，只把"用户说接受 → 移到当前结论"这种动作转译成文件操作
- **数据安全**：每处理 N 项 commit 一次，避免中途 agent crash 丢失进度
- **向 sink 透明**：处理完成后，下次 sink 看到的就是干净的 page，不需要特殊兼容

---

## 输入

用户应提供 `{data_dir}`。如果没有：

1. 先看当前目录是否有 `config.yaml`
2. 否则问用户提供

---

## 主流程

### Step 1: 读取 inbox/pending-review.md

读 `{data_dir}/inbox/pending-review.md`，按 `## ` 段头分组。识别两类段：

- **REVIEW 段**（来自 sink）：标题如 `## 2026-04-25 (来自 sink raw/...)`
- **page_type 候选段**：标题为 `## page_type 候选（来自 sink 自动评估）`

> "未来扩展"段是用户手记，不在本 SKILL 处理范围。

提取所有未处理项（行首是 `- [ ]`）。已处理（`- [x]` 或 `✅` 开头）跳过。

如果完全没有未处理项，输出："inbox 干净，没有需要 review 的项。退出。"

### Step 2: 预扫 needs_review pages

**先动态发现 type folder**（与 sink Step 1、health 4.1 用同一份解析逻辑）：

1. 读 `{data_dir}/config.yaml` 拿 kit_path 和 enabled presets
2. 扫描 `{kit_path}/presets/<preset>/*.yaml`，按统一约定 `yaml.folder ?? f"{name}s"` 解析每个 type 的 folder
3. 合并 vault overrides（MVP 阶段强制空，按 page-type-spec 契约）

得到 `FOLDERS = [folder for type in 动态列表]`。

然后扫描 `{data_dir}/<folder>/*.md`（遍历所有 FOLDERS，**不仅**是 decisions/meetings），找出 `review_status: needs_review` 的 page。

把 inbox 项按"指向哪个 page"分组——同一个 page 的所有 inbox 项一起处理（避免多次打开同一个 page 上下文）。

如果有 page 标了 `needs_review` 但 inbox 里没对应项，输出 WARN：「{page} 标 needs_review 但 inbox 无相关项，可能是历史 sink 漏写。是否要把它的 ## 待确认事实 段交给用户处理？」

### Step 3: 逐 page 处理

#### 3.0 处理顺序

- **多 page 间**：默认按 `{folder}/{filename}` 路径字母序（动态 folder 列表已包含所有受管理 type，不只 decisions/meetings）。允许用户开头说"先处理 X page"插队
- **单 page 内**：**按 inbox 时间倒序**——最新 sink 项先处理。这避免新项处理时遗漏"老项已被新事实覆盖"的情况，并让自动 moot 检测（见 3.2.5）能减少向用户提问

对每个有未处理 inbox 项的 page，依次：

#### 3.1 给用户上下文

```
========================================
处理 page: decisions/嘉宝棋牌室硬件接入方案.md
review_status: needs_review
未处理 inbox 项（3 条）：
  [a] 「3 万预算（硬件 2 万 + 美团接入开发不收费）」是否对外承诺给王五？
  [b] 「6 周内出 v1 demo」是对外承诺，需要确认是否进我的项目排期。
  [c] status 已改为 superseded，「关键约束」段提到的"3 万预算"承诺是否要主动告诉相关方？

页面相关段落：
  ## 待确认事实
  - 原「3 万预算（硬件 2 万 + 美团接入开发不收费）」是否曾构成对外承诺，以及方案作废后是否需要主动通知相关方。
  - 原「6 周内出 v1 demo / 5 月初第一版 demo」是否曾进入个人或团队排期，以及是否需要从计划中移除。
========================================
```

#### 3.2 对每条 inbox 项让用户选择

##### 3.2.1 先分类（按稳定前缀 → 自然语言）

每条 inbox 项处理前先判类型，决定走哪个选择菜单。**判别按以下优先级**（高优先级命中即用，不再继续判）：

| 优先级 | 类型 | 判别 | 菜单 |
|---|---|---|---|
| 1 | **D follow_ups 失效** | 描述以稳定前缀 `[follow_ups_obsolete]` 开头 | 直接走 3.2.6 特殊路径 |
| 2 | **A 事实项** | 描述与 page 当前 `## 待确认事实` 段某行做子串匹配命中（容忍轻微改写） | 走 3.2.2 六选一菜单 |
| 3 | **B 动作项** | 描述含"是否要做/是否进 todo/是否进排期/是否告诉/是否走 X 流程"等动作语 | 走 3.2.3 三选一菜单 |
| 4 | **C 候选项** | 描述含"是否需要建独立 X page/是否值得建档/是否要建 X type" | 路由到 Step 4 page_type 候选段处理（如果 inbox 项当前在 REVIEW 段，迁移到候选段并保留原文） |

**兜底**：若四类都不命中，提示用户"无法自动分类，是 D/A/B/C？"。

**为什么优先级 1 是稳定前缀而非自然语言**：sink 写 inbox 时同一语义可能用不同表述（"全部因方案作废而失效"/"follow_ups 因方案取代可能失效"），靠语义匹配会漏判，把失效待办错误地按事实处理移进会议要点。前缀 `[follow_ups_obsolete]` 是 sink Step 3c 唯一可靠的 routing token。

##### 3.2.2 A 类（事实项）菜单

```
[a] 「3 万预算」是否对外承诺给王五？
选择：
  1) 接受为事实（移入 page compiled 区，按 status 路由——见 3.2.4）
  2) 修改后接受（你给出修改文本）
  3) 跳过（保留 needs_review，下次再 review）
  4) 否决（事实不成立，从 page 删除并加 [否决] 注记）
  5) 后续要追踪（不进 page，进 todo / 下次会议议程）
  6) 已 moot（事实成立但方案已废，不需要 action）
```

| 选择 | 动作 |
|---|---|
| 1 接受 | 把 `## 待确认事实` 中对应那一行**移入** compiled 区（位置见 3.2.4）；inbox 项标 ✅ |
| 2 修改 | 让用户输入修改文本；用修改后的文本替换并移入 compiled 区；inbox 项标 ✅ |
| 3 跳过 | 不动 page；inbox 项也不动 |
| 4 否决 | 从 `## 待确认事实` 段**删除**对应行；inbox 项加 `[否决] {原文}: {用户给的理由}` |
| 5 追踪 | 从 `## 待确认事实` 删除；inbox 项标 ✅；额外提示用户"已记下，但 review skill 不管理 todo，请用你的提醒系统跟进" |
| 6 已 moot | 从 `## 待确认事实` 删除；inbox 项加 `[moot] {原文}: 方案已废，不再 action`；不进任何 compiled 段 |

##### 3.2.3 B 类（动作项）菜单

```
[b] 「6 周 demo」是否进我的项目排期？
选择：
  1) 已执行 / 已决定（说明你做了什么，例：已进 todo / 已告诉对方 / 决定不做）
  2) 后续追踪（请用你自己的 todo 系统记录，本 skill 不管）
  3) 跳过（保留在 inbox，下次再决定）
```

| 选择 | 动作 |
|---|---|
| 1 已执行 | inbox 项加 `✅ {原文}: {用户简短说明}`，**不动 page** |
| 2 追踪 | inbox 项标 ✅；提示"已记下，请用你的 todo 系统跟进" |
| 3 跳过 | inbox 项不动 |

B 类**不动 page**——动作项不在 page 上有对应行，page 状态不变。

##### 3.2.4 A 类"接受"的写入位置（按 type 分流）

A 类选择 1 或 2（接受/修改后接受）的写入位置先按 page 的 **type** 分流，再在 type 内按 status（如有）细分：

**decision**（有 `status` 字段）：

| status | 移入位置 | 时间线 |
|---|---|---|
| `active` | `## 当前结论` | 不需特殊标记 |
| `superseded` | `## 历史确认`（不存在则创建）+ 条目末尾标注"（已超期，仅作历史记录）" | 时间线追加：`- {today} review 确认 {一句话事实}（page 已 superseded）→ inbox/pending-review.md` |
| `deferred` | `## 当前结论` + 内联标记"（方案当前 deferred）" | 不需特殊标记 |

**meeting**（无 `status` 字段）：

- 永远移入 `## 会议要点` 段尾部（追加，不改写已有行——这是 meeting.yaml 例外 2 规定的）
- 时间线不需特殊标记

**其他 type（如 migrate 新增的 merchant-profile 等，无 `status` 字段）**：

| 检查顺序 | 路由位置 |
|---|---|
| 1. yaml description 显式声明的"主要 compiled 段"（如 `## 当前画像`） | 移入该段尾部 |
| 2. yaml 没声明 → page 当前的**第一个 H2 段**（不含 `## 待确认事实` / `## 演化时间线`） | 移入第一个 H2 段尾部 |
| 3. page 完全没有合规 H2 → 创建新段 `## 当前结论`，移入 | 创建并移入 |

> **设计原则**：A 类接受需要"把待确认事实变成确定结论"，所以一定要落在 compiled 区。decision 因 superseded 引入了双层 compiled（`当前结论` + `历史确认`），meeting 只有一层（`会议要点`），其他 type 默认按"第一个 H2"兜底——agent 在没有 yaml 显式约定时不应该硬猜段名，而是用结构化 fallback。
>
> **status 不是通用字段**：之前版本错把"按 status 路由"写成所有 type 都遵守的规则。实际只有 decision 有 status；meeting 和未来新 type 不要硬塞这个字段，也不要走 status-routing 逻辑。

> **为什么 superseded 不进 `## 当前结论`**：superseded page 的「当前结论」是历史快照，反映的是当时的决策状态。继续往里追加新事实会让读取侧把 review 后追加的事实当成"原方案当时就这么定的"，混淆历史。`## 历史确认` 段把这两类信息分开。

##### 3.2.5 处理后的自动 moot 检测（跨项依赖）

每处理完一项，立即扫描本 page **剩余未处理 inbox 项**，做跨项依赖识别。两类情形触发自动 moot：

1. **本项把 follow_ups 移到 obsolete_follow_ups**（来自 follow_ups 失效特殊路径）：
   - 扫描剩余 inbox 项描述，若包含被移项的 follow_up 文本（按子串匹配），自动标 `[moot] {原文}（已被本轮 review 处理：follow_up 已移到 obsolete_follow_ups）`
   - 不再询问用户

2. **本项从 `## 待确认事实` 段删除了某行**（否决/追踪/已 moot）：
   - 扫描剩余 inbox 项描述，若与被删行做子串匹配命中（容忍改写），自动标 `[moot]（已被前一项处理）`
   - 不再询问用户

**例**：处理 [g]「follow_ups 因方案取代失效」时把"硬件清单"移到 obsolete_follow_ups。下一项 [h]「follow_up 硬件清单进 todo」（B 类动作项）描述含"硬件清单"——子串匹配命中，自动标 [moot]，跳过 B 类菜单提问。

##### 3.2.6 特殊处理：follow_ups 失效项

**路由判别（按稳定前缀，不靠自然语言）**：

如果 inbox 项形如 `- [ ] [follow_ups_obsolete] meetings/<file>.md: ...`，按本节特殊路径处理。这个前缀由 sink Step 3c 在反向闭环时显式写入，是 review skill 路由的**唯一可靠依据**。

**legacy 兼容**：MVP 之前的 inbox 项可能用自然语言（"全部因方案作废而失效" / "follow_ups 因方案取代可能失效"等）描述同一事情但缺前缀。这种 legacy 项 fallback 进 A 类按事实处理；用户处理时如发现实际是 follow_ups 失效，可手动选 1 接受后由 agent 临时走本节路径，并提示"建议给老 inbox 项手动加 [follow_ups_obsolete] 前缀以便下次稳定路由"。

**特殊路径动作**（命中前缀的项，跳过分类直接走此路径）：

- 不进 A 类菜单，**不直接接受**
- 而是让 agent 问用户："follow_ups 数组里这 N 条，哪些已失效？哪些仍有效？逐条标 obsolete 或 keep"
- 对每条标 obsolete 的，从 frontmatter `follow_ups` 数组**移除**，加到 `obsolete_follow_ups` 数组（创建该字段如果不存在）：
  ```yaml
  obsolete_follow_ups:
    - text: 我下周一前给王五硬件清单
      obsolete_reason: 方案 superseded by 多店统一 SaaS 平台
      obsolete_date: 2026-05-08
  ```
- 对每条标 keep 的，留在 `follow_ups` 数组
- inbox 项标 ✅
- 触发 3.2.5 自动 moot 扫描

#### 3.3 调整 review_status

处理完该 page 的所有 inbox 项后，检查 page 状态：

- 如果 `## 待确认事实` 段已为空（没有未处理事实）→ 删除该段，并把 `review_status` 改回 `clean`
- 如果还有事实（用户跳过了某些）→ 保留 `## 待确认事实` 段和 `review_status: needs_review`

#### 3.4 写时间线

在 `## 演化时间线` 末尾追加一行：

```
- {today} review 已处理（接受 X / 否决 Y / 追踪 Z）→ inbox/pending-review.md
```

#### 3.5 阶段性 commit

每处理完 1 个 page，做一次 commit，避免中途崩溃丢进度：

```bash
cd {data_dir}
git add <处理的 page> inbox/pending-review.md
git diff --cached --quiet || git commit -m "Review: process pending items on <page>"
```

### Step 4: 处理 page_type 候选段

如果 inbox 里有「page_type 候选」段，单独跑一轮：

对每条候选项问用户：

```
候选：[置信度: 强] 提议新建 `merchant-profile`
  触发信号：塞不下 + 复发
  ...
选择：
  1) 接受 → 创建 yaml，走 migrate add-type 流程
  2) 改成加字段 → 走 migrate add-optional-field 流程
  3) 暂缓 → 保留候选不动
  4) 否决 → 删除并加 [否决] 注记
```

根据选择路由：

- 1：提示用户「我会调用 migrate skill 的场景 1 子流程。请先在 `presets/<preset>/` 下放好 `<name>.yaml`，然后说 '跑一下 migrate'。」
- 2：提示用户「请改 `presets/<preset>/<existing-type>.yaml` 把字段加到「可选」段，然后说 '跑一下 migrate'。」
- 3：保留 inbox 行不动
- 4：删除 inbox 行，加 `- [否决] {type-name}: {理由}` 注记，避免后续 sink 反复评估同一概念

### Step 5: 总结输出

```
========================================
Review 完成

处理统计：
- 接受：N 项
- 修改后接受：M 项
- 否决：K 项
- 跟踪：J 项
- 跳过（保留 needs_review）：L 项

page 状态变化：
- decisions/嘉宝棋牌室硬件接入方案.md: needs_review → clean
- meetings/2026-04-25-王五.md: needs_review（仍有 1 项跳过）

page_type 候选：
- merchant-profile: 接受 → 请按提示跑 migrate
- decision 加 commercial_model: 暂缓

提交：N 个 commit
========================================
```

---

## 异常处理

- **page 不存在但 inbox 引用**：输出 WARN，让用户决定是删 inbox 项还是修复 page 路径
- **`## 待确认事实` 段不存在但 page 标 needs_review**：让用户决定是直接 review_status: clean 还是去 page 里手动加段
- **inbox 项格式异常**（无法识别 page 路径）：跳过该项，输出 WARN 让用户手动看
- **用户中途想退出**：保存进度（已处理项打 ✅、未处理项不动），下次跑 review 接着来

---

## 不在本 SKILL 范围

- **新建/修改 page 的内容**——只移动文本，不重写
- **判断事实真伪**——本 SKILL 永远问用户，不自己决定
- **跨 page 传播变更**——本 SKILL 只动当前 page；如果接受的事实需要影响其他 page，由用户在下次 sink 时通过 raw 触发

---

## 与其他 SKILL 的关系

| SKILL | 关系 |
|---|---|
| `presets/<preset>/SKILL.md` (sink) | sink 写入 `## 待确认事实` 和 inbox 项；review 是这些 inbox 项的唯一处理出口 |
| `migrate/SKILL.md` | review 处理 page_type 候选时，把决策路由到 migrate；不直接修改 yaml |
| `health/SKILL.md` | health 检查到"page 标 needs_review 但 inbox 无对应项"时，建议用户跑 review 排查 |
