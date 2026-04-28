# wiki-mem-kit Health Skill

> **触发条件**：用户说类似：
> - "帮我检查一下 wiki-mem vault"
> - "跑一次 wiki-mem health check"
> - "检查业务记忆有没有坏链 / schema 问题"
>
> **目标**：只读检查，不自动修改文件。发现问题后按 P0/P1/P2 输出。

---

## 输入

用户应提供 `{data_dir}`。如果没有提供：

1. 先看当前目录是否有 `config.yaml`
2. 如果没有，问用户提供数据目录路径

---

## 检查清单

### 1. config.yaml

读取 `{data_dir}/config.yaml`，检查：

- 必须能被 YAML 解析
- 必须有 `schema_version: 1`
- 必须有 `kit_path`
- 必须有 `presets`，且包含 `professional-work`
- 必须有 `data_root`
- 必须有 `overrides.page_types`、`overrides.skill_additions`、`overrides.frontmatter_overrides`
- MVP 强制契约：`overrides.page_types == []`、`overrides.skill_additions == []`、`overrides.frontmatter_overrides == {}`（详见 docs/page-type-spec.md）。非空 → P1 警告"未来字段被提前使用"
- 不应使用旧字段 `kit`

问题分级：
- P0：YAML 无法解析、缺 `schema_version`、缺 `kit_path`
- P1：缺 overrides 子字段、使用旧字段 `kit`

### 2. 目录结构

检查以下目录存在：

- `raw/`
- `decisions/`
- `meetings/`
- `inbox/`

问题分级：
- P0：`raw/` 缺失
- P1：`decisions/`、`meetings/`、`inbox/` 缺失

### 3. git 与 .gitignore

在 `{data_dir}` 下检查 git 是否真正在追踪业务 markdown。**注意**：`git status --short` 不会列已跟踪且干净的文件，所以不能用它判断"文件是否被 git 管"。正确组合是 `git ls-files` + `test -f` + `git check-ignore`。

**前置依赖**：先按 4.1 解析出动态 type folder 列表（`FOLDERS`），然后**分两类**构造检查目标——精确路径和 glob 路径必须分开处理（精确路径直接 check，glob 需要先展开+判断匹配）。混在一起循环会让精确路径被"glob 未匹配"误判逻辑跳过（这是真实 bug 的修复）。

#### 3.1 算法（agent 通用，不依赖 shell）

```
EXACT_TARGETS = ["inbox/pending-review.md"]
GLOB_TARGETS  = ["raw/*.md"] + ["<folder>/*.md" for folder in FOLDERS]

# 1) git 仓库是否初始化
if not exists("{data_dir}/.git"):
    emit("WARN: 未初始化 git")

# 2) 精确路径：直接三连检查
for file in EXACT_TARGETS:
    if not exists(file):
        emit(f"WARN: {file} 不存在")
        continue
    if not git_tracked(file):       # git ls-files --error-unmatch <file>
        emit(f"WARN: {file} 存在但未被 git 跟踪")
    if git_ignored(file):           # git check-ignore -q <file>
        emit(f"FATAL: {file} 被 .gitignore 吞掉")

# 3) glob 路径：枚举展开，未匹配则跳过整个 pattern
for pattern in GLOB_TARGETS:
    files = expand_glob(pattern)    # 返回 [] 如果没匹配
    if files == []:
        continue
    for file in files:
        # 同上三连
        ...
```

**任何能读 SKILL 的 agent 都能按这段算法执行**——不依赖具体 shell 类型。

#### 3.2 参考 bash 实现（bash 4+，macOS 默认 zsh 用户请用 `bash health-check.sh` 或 `bash -c '<paste>'` 执行）

```bash
#!/usr/bin/env bash
set -u
# 用法：cd {data_dir} && bash <(curl -s ...)  或保存为脚本

# 4.1 输入（从 config.yaml 解析后传入；这里写死示例）
FOLDERS=("decisions" "meetings")  # 实际由 4.1 动态生成

EXACT_TARGETS=("inbox/pending-review.md")
GLOB_TARGETS=("raw/*.md")
for folder in "${FOLDERS[@]}"; do
  GLOB_TARGETS+=("${folder}/*.md")
done

# 1) git 仓库是否初始化
if [ ! -d .git ]; then
  echo "WARN: 未初始化 git"
fi

# 2) 精确路径：直接三连检查
for file in "${EXACT_TARGETS[@]}"; do
  if [ ! -f "$file" ]; then
    echo "WARN: $file 不存在"
    continue
  fi
  if ! git ls-files --error-unmatch "$file" >/dev/null 2>&1; then
    echo "WARN: $file 存在但未被 git 跟踪"
  fi
  if git check-ignore -q "$file"; then
    echo "FATAL: $file 被 .gitignore 吞掉"
  fi
done

# 3) glob 路径：用 nullglob 展开，未匹配则跳过
shopt -s nullglob
for pattern in "${GLOB_TARGETS[@]}"; do
  files=( $pattern )
  for file in "${files[@]}"; do
    if [ ! -f "$file" ]; then continue; fi
    if ! git ls-files --error-unmatch "$file" >/dev/null 2>&1; then
      echo "WARN: $file 存在但未被 git 跟踪"
    fi
    if git check-ignore -q "$file"; then
      echo "FATAL: $file 被 .gitignore 吞掉"
    fi
  done
done
shopt -u nullglob
```

**zsh 等价提示**：zsh 没有 `shopt -s nullglob`，等价是 `setopt NULL_GLOB`（局部启用用 `(N)` 修饰）。但我们建议直接用 bash 跑这段——agent 一般有 bash 可用，无需改写为 zsh。

**为什么不能混在一起**：bash 中 `for x in raw/*.md` 在 raw/ 没文件时 `x` 取字面值 `"raw/*.md"`，可以用 `[ "$x" = "$pattern" ]` 检测。但精确路径 `inbox/pending-review.md` 没有 `*` 字符，`for x in inbox/pending-review.md` 就一个值，自然等于 pattern——这时不能 continue，否则精确文件永远不会被检查（这是一个真实 bug，曾让 inbox 被 ignore 的情况漏掉）。

判断：
- `git check-ignore` 命中（退出码 0）= `.gitignore` 正在吞业务 markdown，**P0**
- `git ls-files --error-unmatch` 失败（退出码 ≠ 0）但文件存在 = 用户忘了 `git add`，**P1**
- `.git` 目录不存在 = vault 没初始化 git，**P1**

问题分级：
- P0：`.gitignore` 忽略 `raw/*.md`、动态发现的任一 type folder 下的 `*.md`、或 `inbox/pending-review.md`
- P1：未初始化 git，或文件存在但未跟踪
- P1：动态 type 列表中某 folder 不存在（migrate 没创建或被手动删除）

### 4. page frontmatter

#### 4.1 动态发现 page_type 与目录

不再硬编码 `decisions/` 和 `meetings/`。从 config + preset yaml 反推所有受管理的 type：

```bash
# 1) 读 {data_dir}/config.yaml 拿 kit_path 和启用的 presets
# 2) 对每个 preset：扫描 {kit_path}/presets/<preset>/*.yaml
# 3) 每个 yaml 解析 folder（按 page-type-spec 的统一约定）：
#    FOLDER = yaml.folder if present else f"{yaml.name}s"
# 4) 合并 config.yaml 的 overrides.page_types 私有 type
#    MVP 阶段：本字段强制为空数组（见 page-type-spec），实际为 noop
#    Future：按 [{path: <yaml 路径>}] 格式合并；kit preset 同名 type 优先
```

得到一份**动态 type 列表**：`[{name, folder, source: kit/preset|vault/override}]`，folder 字段已经按统一规则解析完毕。

扫描时遍历这份列表，对每个 folder 下的 *.md 应用 4.2 通用检查；如果 type 是已知的（decision / meeting），额外应用 4.3 type-specific 检查。

#### 4.2 通用 frontmatter 检查（应用到所有 type folder）

所有 page 必须有：

- `title`
- `type`（且必须等于 yaml 的 name）
- `schema_version: 1`
- `sources`（数组，至少 1 项）
- `review_status: clean | needs_review`

问题分级：
- P0：缺 `schema_version`、`sources`、`review_status`、`type`、`title`
- P0：`type` 字段值与文件所在 folder 对应的 yaml `name` 不匹配（说明 page 放错目录或 frontmatter 写错）

#### 4.3 type-specific 检查（仅 decision / meeting）

decision 额外必须有：
- `status: active | superseded | deferred`
- `aliases`，且至少 1 个

meeting 额外必须有：
- `attendees`
- `meeting_date`
- `obsolete_follow_ups` 项（如存在）必须是对象数组，每项含 `text`、`obsolete_reason`、`obsolete_date`

问题分级：
- P1：decision 缺 `status` / `aliases`，meeting 缺 `attendees` / `meeting_date`
- P1：meeting 的 `obsolete_follow_ups` 项缺必填子字段

#### 4.4 未知 type 的 fallback

如果 4.1 发现的 type 名不是 `decision` 或 `meeting`（如未来用户加了 `merchant-profile`），输出 INFO：

```
INFO: type "<name>" 仅做通用 frontmatter 检查；type-specific 字段检查未实现。
   如需深度检查，请在 health/SKILL.md 里加对应规则，或等 future 版本支持从 yaml description 自动解析必填字段。
```

不阻塞 health 整体执行。新 type 的 page 至少有通用检查兜底。

### 4.5 page_type yaml

读取 `kit_path/presets/professional-work/*.yaml`，检查每个 page_type：

- 必须有 `name`
- 必须有 `schema_version: 1`
- 必须有 `description`
- description 必须说明必填 frontmatter、更新规则和 REVIEW 触发

问题分级：
- P0：缺 `schema_version` 或 `description`
- P1：description 缺关键段落

### 5. review_status 语义

检查：

- 如果 page 有 `## 待确认事实`，frontmatter 必须是 `review_status: needs_review`
- 如果 `review_status: needs_review`，page 应该有 `## 待确认事实` 或 inbox 中有对应 REVIEW 项
- 金额、预算、报价、交付时间、对外承诺出现在「当前结论」或「会议要点」里时，检查是否同时有 `review_status: needs_review`
- meeting 上 `obsolete_follow_ups` 里有项，但 `follow_ups` 数组里仍残留同文本的"活跃"项 → 数据污染

**review 积压检测**（基于 inbox 时间戳，不依赖 page 自身字段）：

- 统计 `review_status: needs_review` 的 page 数量
- 解析 `inbox/pending-review.md`，按 `## YYYY-MM-DD ...` 段头读出每段日期；找到**最老**的还含 `- [ ]` 未处理项的段日期
- 计算"最老未处理段日期"距今天数；如果 > 14 天且未处理项总数 > 5，输出 P1 提醒

> 这里不检测每个 page 的 review_since 时间戳——page frontmatter 没这字段，引入会增加 schema 负担。inbox 段日期是 sink 时确定写下的，足以反映"积压时长"信号。
>
> 如果某个段所有项都已 ✅，跳过该段（按它"已经处理过"算）；遍历到下一段。

问题分级：
- P0：高风险事实以确定语气进入 compiled 区，但 `review_status: clean`
- P0：`obsolete_follow_ups` 与 `follow_ups` 内容重复（活跃 + 已废视为同一文本，违反读取契约）
- P1：`review_status: needs_review` 但找不到待确认事实或 inbox 项
- P1：inbox 最老未处理段日期距今 > 14 天 且未处理项 > 5 → 建议跑 `review/SKILL.md` 处理

### 6. sources 存在性

对每个 page 的 `sources`：

- 路径必须相对 `{data_dir}`
- 文件必须存在
- replacement decision 不应为了继承历史复制旧 page 的 sources；继承关系应使用 `supersedes`

问题分级：
- P0：source 文件不存在
- P1：明显把旧方案 sources 复制到新 replacement decision

### 7. wikilink 基础检查

扫描正文中的 `[[...]]`：

- `[[raw/...]]` 应指向存在的 raw 文件（允许省略 `.md`）
- 非 raw wikilink 应能通过文件名、frontmatter title、aliases 至少一种方式找到 page
- 多个 page 命中同一 wikilink 时报警

问题分级：
- P1：wikilink 找不到目标
- P2：wikilink 命中多个目标

### 8. superseded 关系

检查：

- `status: superseded` 的 decision 必须有 `superseded_by`
- 有 `superseded_by` 的 page 正文顶部必须有 superseded block
- 有 `supersedes` 的新 page，其目标旧 page 应存在，并最好反向包含 `superseded_by`

问题分级：
- P1：单向 supersede 缺反链
- P1：superseded page 没有顶部 block

---

## 输出格式

按以下格式输出：

```markdown
## Health Check Result

P0:
- <文件>: <问题>。建议：<修法>

P1:
- <文件>: <问题>。建议：<修法>

P2:
- <文件>: <问题>。建议：<修法>

Summary:
- pages scanned: N
- raw files scanned: M
- broken sources: K
- broken wikilinks: L
```

如果没有问题，明确说：

> 未发现 P0/P1/P2 问题。当前 vault 通过 health check。

---

## 约束

- 本 skill 只读，不直接改文件
- 不要把 P2 包装成阻塞问题
- 如果需要修复，输出建议后等用户明确要求再修改
