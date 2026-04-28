# wiki-mem-kit Migrate Skill

> **触发条件**：用户对 page_type 做了变更（改 yaml）后，对 agent 说类似：
> - "我加了一个新 type，帮我跑一下迁移"
> - "我把 decision 加了 mentions_people 字段，帮我看看老的怎么办"
> - "page_type 升级了，帮我同步"
>
> **agent 兼容性**：本流程只读写文件，对 agent 通用。
>
> **MVP 范围**：本 SKILL 只实现 6 种场景中**可加性**的 3 种：
> - 场景 1: 新增 page_type
> - 场景 2: 给已有 type 加可选字段
> - 场景 3: 给已有 type 加必填字段
>
> 场景 4/5/6（拆 / 合 / 删）请参考 `docs/type-evolution.md` 手动操作。

---

## 入口：识别变更场景

先区分变更发生在哪个边界：

- **kit preset 变更**：改的是 `{kit_path}/presets/.../*.yaml`。这是上游规则升级，提交应发生在 kit 仓库。
- **vault 私有覆盖变更**：改的是 `{data_dir}/config.yaml` 的 `overrides.page_types` 或私有 page_type 文件。提交应发生在数据仓库。**MVP 阶段不支持**——`overrides.page_types` 强制空数组（见 page-type-spec）；用户若改了该字段，本 SKILL 拒绝执行并提示"vault 私有 type 是 future 功能"。

MVP 默认只支持 kit preset 变更。如果用户改的是 vault 私有覆盖，先输出当前不支持自动迁移，并建议把变更整理为真实 raw 后再考虑新增 page_type。

执行步骤：

1. 读取 `{data_dir}/config.yaml`，拿到 `kit_path`
2. `cd {kit_path}` 并 `git status`：确认用户刚改了 `presets/.../*.yaml`
3. `cd {data_dir}` 并 `git status`：确认数据仓库当前没有未提交业务改动，避免迁移和用户手改混在一起
4. **强制提示**：「迁移涉及两个仓库各自的提交：
   - **kit repo（`{kit_path}`）**：你的 preset yaml 变更会在这里 commit；
   - **data repo（`{data_dir}`）**：所有 page 回填、folder 创建、inbox 更新会在这里 commit；
   两边历史独立，回滚也独立。开始前先在 data repo 做 pre-migration commit。继续吗？」
5. 用户确认后，在 `{data_dir}` 执行：
   ```bash
   git add -A
   # 干净仓库下 git commit 会因 nothing-to-commit 失败，需先判断
   if git diff --cached --quiet; then
     # 工作区干净，记录 HEAD 给后续 revert 使用，不创建空 commit
     PRE_MIGRATION_HEAD=$(git rev-parse HEAD)
     echo "Pre-migration HEAD: $PRE_MIGRATION_HEAD（无未提交变更，未创建 snapshot commit）"
   else
     git commit -m "Pre-migration snapshot"
   fi
   ```
   把 `PRE_MIGRATION_HEAD` 或 snapshot commit 的 SHA 记下来，迁移失败时用 `git reset --hard <sha>` 回滚。
6. 读 `{kit_path}` 里改过的 yaml，比对 kit 仓库 HEAD~1 同名文件，识别变更类型：
   - 新增了一个 `<new>.yaml` → **场景 1**
   - 现有 yaml description「可选」段加了字段 → **场景 2**
   - 现有 yaml description「必填」段加了字段 → **场景 3**
   - 其他（删字段、改字段名、改 type 名等）→ **场景 4/5/6**：停止，输出 `docs/type-evolution.md` 链接，让用户手动处理
7. 按对应子流程执行

---

## 子流程 A: 场景 1 - 新增 page_type

**前提**：用户已经在 `presets/<preset>/` 下放好新的 `<new>.yaml`。

**变更分布**：新 yaml 在 **kit repo**；新 folder 在 **data repo**。两边各自 commit。

**步骤**：

1. 验证 yaml 合法（必须有 `name` 和 `description`）
2. 验证 description 包含 5 段（是什么 / 必填 frontmatter / 何时归到这里 / 更新规则 / REVIEW 触发）
   - 不全的话报警，告诉用户哪段缺，但不阻塞
3. 解析 folder（按 page-type-spec 的统一解析约定）：
   ```
   FOLDER = yaml.folder if yaml.folder else f"{yaml.name}s"
   ```
4. 创建 `{data_dir}/$FOLDER/` 目录
5. **不需要回填**：旧 raw 不会自动重新 sink 到新 type，但下次 sink 新 raw 时模型会按新 type 描述判断
6. 提示用户：「新 type `<name>` 已可用，目录 `$FOLDER/`。如果想让它生效在历史 raw 上，告诉我：'重新 sink raw/<某文件>'」
7. **kit repo 提交**（yaml 变更）：
   ```bash
   cd {kit_path}
   git add presets/<preset>/<name>.yaml
   git commit -m "presets/<preset>: add page_type <name>"
   ```
8. **data repo 提交**（folder 创建）：
   ```bash
   cd {data_dir}
   touch "$FOLDER/.gitkeep"
   git add "$FOLDER/.gitkeep"
   git diff --cached --quiet || git commit -m "Add folder $FOLDER for new page_type: <name>"
   ```
   （新 folder 没有内容时 git 默认不跟踪，先放一个 `.gitkeep`；首条 sink 写入后可删除该文件）
9. 提示用户：「kit repo 的 yaml 变更已提交；data repo 的 folder `$FOLDER/` 已就绪。两边历史独立，回滚分开做。」

---

## 子流程 B: 场景 2 - 加可选字段

**前提**：用户改了 yaml 的 description「可选」段，加了一个新字段。

**变更分布**：yaml 在 **kit repo**；inbox 提醒在 **data repo**。两边各自 commit。

**步骤**：

1. 把字段名 + 加入日期写到 `{data_dir}/inbox/pending-review.md` 的「优化建议」段：

   ```markdown
   ## 2026-04-26 type 升级
   - [ ] decision 加了可选字段 `mentions_people`。下次访问已有 page 时如果合适，请补上。
   ```

2. **不批量回填**：可选字段缺失不算违规，强行回填会引入低质量数据
3. 不升 schema_version（可选字段变更不算 schema 破坏）
4. **kit repo 提交**：
   ```bash
   cd {kit_path}
   git add presets/<preset>/<type>.yaml
   git commit -m "presets/<preset>: add optional field <field> to <type>"
   ```
5. **data repo 提交**：
   ```bash
   cd {data_dir}
   git add inbox/pending-review.md
   git commit -m "Note new optional field <field> for lazy backfill"
   ```

---

## 子流程 C: 场景 3 - 加必填字段

**前提**：用户改了 yaml 的 description「必填」段，加了一个新字段。

**风险提示**：必填字段变更 = schema 升级。所有老 page 不合规，需要批量回填。

**变更分布**：yaml + schema_version 在 **kit repo**；批量回填 + inbox 在 **data repo**。两边各自 commit。

**步骤**：

### Step C1: 提示并升 schema_version

提示用户：「这次是必填字段变更，意味着所有现有 `<type>` page 都需要回填这个字段。我会：
1. 把 type 的 schema_version 从 N 升到 N+1（kit repo 提交）
2. 遍历所有该 type 的 page，对每个 schema_version < N+1 的 page 做回填（data repo 提交）
3. 推断不出的列入 inbox/pending-review.md 让你手动补
继续吗？」

用户确认后：

- 在 yaml 顶部加（或更新）`schema_version: N+1` 字段
- （如果 yaml 还没有 schema_version，初始化为 1，并把"加 N+1 字段"那条算作升到 2）
- **kit repo 提交**：
  ```bash
  cd {kit_path}
  git add presets/<preset>/<type>.yaml
  git commit -m "presets/<preset>: bump <type> to schema v{N+1} (require <field>)"
  ```

### Step C2: 遍历回填

对 `{data_dir}/<folder>/*.md` 中每个该 type 的 page：

1. 读 frontmatter 的 schema_version（缺失视作 1）
2. 如果 schema_version >= 新版本号，跳过
3. 否则尝试推断字段值：
   - 从 page 的 sources 数组里找到对应的 raw 文件
   - 用模型按新字段语义从 raw 里抽取
   - 抽不出 → REVIEW
   - 抽出 → 更新 frontmatter，schema_version 升到新版本号
4. 在 page 「演化时间线」末尾加一行：`- {today} schema 升级到 v{N+1}，回填字段 <field>`

### Step C3: 输出统计 + REVIEW

```
迁移完成：
- <type> page 总数：M
- 自动回填成功：K
- 待人工补：M - K（已写入 inbox/pending-review.md）
```

### Step C4: data repo 提交

```bash
cd {data_dir}
git add -A
git diff --cached --quiet || git commit -m "Migrate <type> to schema v{N+1}: backfill <field>"
```

（kit repo 提交已在 Step C1 完成，本步只 commit 回填产物。）

---

## REVIEW 持久化

所有迁移过程中产生的 REVIEW 都写到 `inbox/pending-review.md`，格式与 sink 流程一致：

```markdown
## 2026-04-26 (来自 migrate add-required-field decision.decision_date)

- [ ] decisions/嘉宝棋牌室硬件接入方案.md: 推断 decision_date 失败（raw 文件没明确日期），请补
```

---

## 异常处理

- **git 不干净**：用户没 commit 旧改动 → 拒绝执行，让用户先 commit / stash
- **yaml 改错了**：必填字段写在错的段、description 缺段 → 报告位置，回滚（git restore）
- **回填中模型反复失败**：某 page 连续 3 次抽不出字段 → 标 REVIEW，跳过，继续下一个

---

## 不在本 SKILL 范围内的（先看 docs/type-evolution.md）

- 拆分 type（场景 4）
- 合并 type（场景 5）
- 删除 type（场景 6）

未来如果用户高频遇到这三种，会迭代到 SKILL 的子流程 D/E/F。
