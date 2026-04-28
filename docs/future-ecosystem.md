# 生态展望

> 这份文档不是 MVP 范围，是给未来的自己 / 贡献者留的方向锚。
> 实际实现前应先验证 MVP 跑通了，且有真实用户反馈支撑这些方向有需求。

---

## 方向 1: Role Picker（场景 + 角色驱动的 preset 推荐）

**问题**：MVP 阶段 setup 时只问"使用场景"（工作 / 个人 / 都要），但同样是工作场景，PM、TPM、AI 全栈、销售、客服需要的 page_type 差别很大。
让用户在 setup 阶段一次性回答太多会劝退；让用户后续慢慢加 type 又错过了"上手就能用"的窗口。

**设计草案**：

```
Setup 第 N 步：「你的角色」picker

┌────────────────────────────────────────────────┐
│ 选择 1-3 个最贴近你日常工作的角色：             │
│                                                │
│ ☐ 产品经理 (PM)                                │
│   → 默认 preset: decision + meeting + spec    │
│ ☐ 技术 PM (TPM)                                │
│   → 默认 preset: decision + meeting + tracker  │
│ ☐ AI 全栈 / 全栈工程师                         │
│   → 默认 preset: decision + meeting + arch     │
│ ☐ B 端商家 BD / 销售                           │
│   → 默认 preset: decision + meeting + merchant │
│ ☐ 客服 / 用户研究                              │
│   → 默认 preset: meeting + user-feedback       │
│ ☐ 自由工作者 / 独立咨询                        │
│   → 默认 preset: decision + meeting + client   │
│ ☐ 学生 / 研究者                                │
│   → 默认 preset: decision + meeting + reading  │
│ ☐ 我的角色不在上面（自定义）                   │
└────────────────────────────────────────────────┘
```

**实现方式**：
- 每个角色对应 `presets/role-bundles/<role>.yaml`，里面声明该角色推荐的 page_type 组合
- setup 流程合并多角色的 page_type 集合（去重）
- 用户后续可以单独装 / 删某个 type

**触发实现的信号**：3+ 个用户反馈"professional-work 不够用，但我又不知道怎么自己加 type"。

---

## 方向 2: AI 反向优化 SKILL（自我迭代）

**问题**：SKILL.md 是人写的，写完之后会过时。但每次 sink 时模型其实在一线 ground truth 上工作，知道哪段描述模糊、哪个判断难做、哪个 REVIEW 触发其实没必要。
这些信号应该回流到 SKILL，而不是消失在聊天里。

**设计草案**：

```
每次 sink 完成后，模型可选输出一段 META block：

META（关于本 SKILL 的反馈，不影响输出）:
- decision.yaml description「拆分原则」第二条不够清晰，
  我在判断"商业模式是同方案的不同切面"时犹豫了 30 秒
- REVIEW 触发「金额数字」太宽，预算 / 销售额 / 用户数 都触发了，
  实际可能只需要"对外报价"触发就够

这些 META 持久化到 inbox/skill-feedback.md，
积累到 N 条后，agent 可以触发 `analyze-skill-feedback` 流程：
- 读所有 feedback
- 聚类共性问题
- 生成 SKILL 修改建议（diff 形式）
- 给用户人工 review，approve 后才改
```

**关键纪律**：
- 反向优化必须人在 loop 里。模型不能自动改自己的 SKILL，否则会走偏
- diff 必须 git commit，可 revert
- 每次改完 SKILL 后跑一遍假数据 regression test

**触发实现的信号**：MVP 跑 1 个月后，用户反馈"sink 输出质量在波动，希望 SKILL 能稳一点"。

---

## 方向 3: Plugin Marketplace 一键安装

**问题**：当前 setup 流程是"用户告诉 agent 装 GitHub 仓库"，依赖用户知道 wiki-mem-kit 这个项目。

**设计草案**：

接入 Cowork / Claude Code 的 plugin marketplace：
- 用户在 marketplace 搜"业务记忆"、"会议沉淀"、"PM 笔记"
- 一键安装，自动触发 setup SKILL
- 后续更新自动推送

**前置依赖**：
- Cowork plugin 提交流程稳定
- wiki-mem-kit 有清晰的 LICENSE 和发布流程

**触发实现的信号**：MVP 验证后，准备扩散到非种子用户。

---

## 方向 4: 跨 vault 的图谱共享（团队 / 社群版）

**问题**：MVP 是单人 vault。但实际工作中，"团队共享一份业务记忆"是合理诉求（如 PM 团队、商家运营团队）。

**风险**：隐私 / 权限 / 合并冲突 / 主观 vs 客观事实的边界

**设计草案（雏形）**：

- 公共 vault + 个人 vault 双层
- 公共 vault 走团队共享 git remote
- 个人 vault 不分享
- sink 时让模型问："这条要进公共还是个人？"
- 引用关系：个人 page 可以 wikilink 到公共 page，反之不行（防个人信息泄露到公共）

**触发实现的信号**：单人版有稳定使用后，团队询问"我们能合用吗"。

**警告**：这个方向会让 wiki-mem-kit 的复杂度量变到质变。先把单人版做好。

---

## 方向 5: 接入语音 / 邮件 / IM 的自动 raw 收集

**问题**：MVP 假设用户会手工把 raw 整理成 markdown 放到 `raw/`。这是巨大的手工成本。

**设计草案（雏形）**：

- raw 收集 adapter：
  - 微信语音 → 转文字 → raw 草稿
  - Gmail / Outlook 邮件 → 提取要点 → raw 草稿
  - 飞书 / 企微会议纪要 → 直接落 raw
- adapter 输出始终是「raw 草稿」，要用户 review 一下再正式入仓库
- adapter 不直接调 sink，给用户在 raw 阶段就有干预空间

**触发实现的信号**：用户反馈"我有材料但懒得整理成 markdown 放进去"。

---

## 不会做的事（明确范围）

- **不做托管 SaaS**：用户的业务记忆是高度敏感数据，wiki-mem-kit 永远是本地优先
- **不做 GUI 客户端**：能用 Obsidian / VS Code 的 markdown preview 就够了，做客户端 = 重新发明 Obsidian
- **不做模型微调**：MCP + SKILL 的灵活性 > 微调的稳定性
- **不做企业版功能**：留给商业化用户基于 MIT 自己 fork
