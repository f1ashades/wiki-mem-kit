# page_type 演化指南

随着用户使用 wiki-mem-kit，page_type 一定会变。这份文档列出 6 种典型变更场景、风险，以及推荐策略。

`migrate/SKILL.md` 实现的是**可加性变更**（场景 1/2/3），其余 3 种由本文档给出操作思路，等用户实际遇到再迭代成 SKILL。

---

## 场景 1: 新增 page_type

**例**：用户已经在用 `decision` 和 `meeting`，发现需要 `merchant-profile` 来记每个商家的状态。

**风险**：低。已有 page 不受影响，新 type 从空开始。

**步骤**：

1. 在 `presets/<preset>/` 下加 `merchant-profile.yaml`，写好 5 段 description
2. 重新走一遍 setup 的 Step 4（让 agent 重新加载 skill），新 type 立即可用
3. 历史 raw 不会自动追溯到新 type，但下次 sink 时模型会看到新 type 描述并据此判断

**MVP 是否实现**：是。`migrate/SKILL.md` 中 `add-type` 子流程。

---

## 场景 2: 给已有 type 加可选字段

**例**：`decision` 原来没 `mentions_people`，现在想加。

**风险**：低。已有 page 缺这个字段不算违规（可选字段），读取代码要兼容缺失值。

**步骤**：

1. 修改对应 `<type>.yaml` 的 description（在「必须的 frontmatter」段把字段加到「可选」一列）
2. 不需要回填。下次 sink 新 raw 时，模型按新版 description 写，老 page 暂时空着
3. （可选）想统一时，在 `inbox/pending-review.md` 列一条提醒，鼓励用户在再次访问老 page 时补字段

**MVP 是否实现**：是。`migrate/SKILL.md` 中 `add-optional-field` 子流程，主要做"提醒"，不强制回填。

---

## 场景 3: 给已有 type 加必填字段

**例**：`decision` 想把 `decision_date` 从可选改成必填。

**风险**：中。老 page 全部不合规，需要回填。

**两种迁移策略**：

### 策略 A：批量回填（推荐用于必填字段）

1. 把字段加到 description 的「必填」段，并把 `schema_version` 在 type 内部记一下（kit 端：在 yaml 里维护一个版本号映射；page 端：每个 page frontmatter 的 `schema_version` 字段）
2. `migrate/SKILL.md` 的 `add-required-field` 子流程：
   - 遍历所有该 type 的 page
   - 找到 schema_version < 新版本号的 page
   - 对每个老 page，从 `sources` 数组的 raw 文件里推断字段值（如 `decision_date` 从 raw 文件名前缀提取）
   - 推断不出的列入 REVIEW，让用户补
   - 完成后把 page 的 `schema_version` 升级
3. 走完后老 page 全部合规

### 策略 B：lazy migration（不推荐用于必填字段，仅在批量推断成本极高时用）

不主动回填，下次有人触发该 page 的更新时再补。
副作用：在补完之前，整个仓库处于"既有合规又有不合规"的状态，可能让搜索 / agent 行为不一致。

**MVP 是否实现**：是。优先策略 A。

---

## 场景 4: 拆分一个 type

**例**：`merchant-profile` 太杂了，想拆成 `merchant-basic`（基础档案） + `merchant-deals`（合作历史）。

**风险**：高。每个老 page 都要决定拆成几个新 page，内容如何分配。

**步骤（手动操作 + 工具辅助）**：

1. 在 yaml 里新增两个 type，旧 type 标 deprecated（不删，保留历史 page 可读）
2. 对每个老 `merchant-profile` page：
   - 用模型按新两个 type 的 description 重新切分内容
   - 切完后人工 review 一遍（高风险变更必须有人在 loop 里）
   - 老 page 文件加 `superseded_by: [新 page 1, 新 page 2]` 字段，移到 `archive/` 目录
3. 全部迁移完后才能删掉老 type 定义

**MVP 是否实现**：否。手动 + 文档指引；未来迭代为 `split-type` 子流程。

---

## 场景 5: 合并两个 type

**例**：`meeting` 和 `1on1` 发现没必要分开，合并到 `meeting`。

**风险**：高。可能有 frontmatter 字段冲突（如 `1on1` 有 `peer_name`，`meeting` 没有）。

**步骤**：

1. 决定保留哪个 type 的 name 作为主名（一般是高频使用的那个）
2. 取两个 type 字段的并集，必填字段中至少一方独有的，需要降级为可选（否则老的合并过来不合规）
3. 把弃用 type 的所有 page 重写 frontmatter（type 改名 + 字段映射）
4. 老 type 定义删除前确认 0 个 page 引用

**MVP 是否实现**：否。同场景 4。

---

## 场景 6: 删除一个 type

**例**：实验发现 `quarterly-okr` 没人用，撤掉。

**风险**：取决于已有数据量。

**两种情况**：

- **0 个 page**：直接删 yaml，无副作用
- **N 个 page**：先决定历史 page 怎么处置：
  - 移到 `archive/quarterly-okr/`，文件本身保留可读
  - 或迁移到其他 type（走场景 4 的逻辑）
  - 或直接删（最不推荐，破坏历史）

**MVP 是否实现**：否。文档指引为主。

---

## 通用安全网：git

不管哪个场景，迁移前后必须有清晰 git commit：

```bash
cd {data_dir}
# Pre-migration snapshot：干净仓库下 nothing-to-commit 会失败，需先判断
git add -A
if git diff --cached --quiet; then
  PRE_HEAD=$(git rev-parse HEAD)  # 干净则只记 HEAD，不创建空 commit
else
  git commit -m "Pre-migration snapshot: scenario X"
fi
# ... 执行迁移 ...
git add -A && git commit -m "Post-migration: scenario X complete"
```

任何迁移翻车都可以 `git reset --hard $PRE_HEAD` 或 `git revert HEAD` 回到上一个状态。

migrate/SKILL.md 的子流程必须在执行前**提示用户先 commit**，并在执行后自动 commit 一次。

---

## 设计原则

回看 6 个场景的共性，我们做了这些设计选择，让 wiki-mem-kit 比"内置数据库 schema 迁移"更容易：

1. **schema 是描述性 yaml + 模型理解，不是强校验代码**——加可选字段不需要数据库 migration
2. **page = 独立 markdown 文件**——拆分 / 合并都是文件操作，不需要 ALTER TABLE
3. **timeline 段保留历史**——任何变更都不会丢"演化路径"
4. **git 是天然 rollback**——不需要自建 undo 系统

代价：场景 4/5/6 仍然需要人在 loop 里。我们认为这是值得的，**自动迁移 = 自动毁灭历史**的风险更高。
