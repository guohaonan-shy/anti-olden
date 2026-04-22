---
name: comm-brief
description: 从飞书群聊/P2P 消息里提炼"变化观察"——新发生了什么、谁表现出什么新特征、群话题有什么变动——候选写入群/人档案。触发：用户说"xx 群最近啥情况"/"最近半个月 xx 群有什么变化"/"今天谁 @ 我了"/"看看跟 X 最近互动"/"帮我梳理今天的消息"。依赖 lark-cli。本 skill 的主业就是沉淀，不只是展示。
allowed-tools: Bash Read Write Edit AskUserQuestion
---

# 沟通简报（消息驱动的 memory 沉淀）

## 你要做什么

**从消息里观察 → 生成候选 memory 更新 → 让用户审阅+改+落盘。**

不是"给用户看漂亮简报"——展示出来的内容本身就是**拟写入 memory 的 diff**。用户看的是"这些观察要不要落档"，不是"总结文章"。

两种输入形态：
- **群动态**（某个 chat_id + 时间窗）→ 冷启动 14 天 / 增量按 checkpoint
- **人物互动**（某个 ou_id + 时间窗）→ 聚焦该人近期表现

两种输出形态都是**增量语气的候选观察**——"过去 14 天新发现 / 新变化 / 模式强化"，不是"这个群长这样"的状态快照。

## 工具权限

- **Bash** — 调 `lark-cli`
- **Read** — 读 `../shared/memory/`（背景上下文）+ 读 `../shared/references/lark-cli-cookbook.md`
- **AskUserQuestion** — 范围确认、消歧、**所有写入确认都走此工具**（不是文本 A/B/C/D）
- **Write / Edit** — 写 `../shared/memory/` 下的 `groups/*.md` 和 `persons/*.md`。**权限边界**和**写入前的 4 选项通用规则**见 `../shared/memory/index.md`

## 前置检查

1. `lark-cli --version` —— 是否安装
2. `lark-cli auth status` —— 是否登录；**顺便拿 `userOpenId`**（`jq -r '.userOpenId'`）供 @ 过滤和"我自己发的"识别

---

## 工作流（对话式）

### Step 1: 确认范围

用户可能说得泛。调 `AskUserQuestion` 确认：
- 今日 @ 汇总（triage 型）
- 某个群的动态 → 再问"是哪个群？最近多久？"
- 跟某人的近期互动 → 再问"是哪个人？"

如果用户已经指定了群名，用 `+chat-search --query "<关键词>"` 消歧后**调 AskUserQuestion 让用户确认**"是 `oc_xxx` 这个群吧"。

### Step 2: 读群/人档案 + 决定窗口

**群路径**：
- glob `../shared/memory/groups/*.md` → 按 frontmatter `chat_id` 匹配
- 命中：Read 整个档案，拿 `last_processed_at` / `last_processed_message_id` / 已有"话题基线"/"近期简报"
- 未命中：进入冷启动

**窗口判断**：

| 条件 | 模式 | 拉取窗口 |
|---|---|---|
| 档案不存在 / 无 `last_processed_at` / 距今 > 30 天 | **冷启动** | 默认最近 **14 天** |
| `last_processed_at` 在 30 天内 | **增量** | `--start = last_processed_at + 1s` |
| 用户显式指定 | **自定义** | 按用户 |

**告诉用户选了哪个模式**（文本说明，不用 AskUserQuestion）：
> "这个群 10 天前处理过，走增量模式，拉 2026-04-12T... 起的消息。"

**人物路径**：类似，glob `persons/*.md`，但人档案没有 checkpoint 字段（目前），默认冷启动 14 天；也可以用关联群的 checkpoint 做增量锚点（Phase 0 可简化为"每次 14 天"，增量以后再加）。

### Step 3: 拉消息（自动分页直到窗口边界）

查 `../shared/references/lark-cli-cookbook.md` "消息读取" + "增量/窗口拉取"配方。

- **群动态**：`+chat-messages-list --chat-id <id> --start <ISO8601> --sort desc`，循环 `page_token` 直到 `has_more=false` 或跨出窗口
- **人物互动**：`+messages-search` 按 sender 过滤（flag 细节先 `--help`）
- **今日 @**：cookbook "仍需 Phase 0 发现"段；短期兜底 = 逐 chat 拉近 24h + `jq` 过滤 `mentions` 含 user `userOpenId`

**hydration 按需**：retrospective 默认**不 hydrate**（图片/reply_to/thread 对趋势画像收益低）。triage 某条消息被遮蔽（reply_to 出窗、有图片）才补。

### Step 4: 读其他 memory（背景上下文）

- Read `user-profile.md`（知道"我"是谁 → 影响"我"在本群的观察角度）
- 涉及的群内成员 → glob `persons/*.md` + frontmatter `lark_user_id` 匹配 → Read（看"过去档案"是什么样，好判断"这次新观察是强化还是冲突"）
- 涉及的群 → group 档案已在 Step 2 Read 过

**Memory 不存在不报错、不中断。**

### Step 5: 生成**变化观察清单**（diff-shape，不是状态快照）

**核心心法**：每条观察都要能直接映射到某个 memory 文件的某个 section。用**"新发现 / 强化 / 冲突 / 退潮"**这种**增量语气**，不用"这个群是什么样"的描述语气。

**格式**（按写入目标分组）：

```
## 候选写入：groups/<pinyin>.md

### `## 群氛围`（新观察 / 强化 / 冲突）
- <观察条目——例："过去 14 天群氛围偏工作吐槽 + brainstorm，跟此前档案'轻松闲聊'的描述有冲突，建议更新">
- ...

### `## 关键人物`（群内成员的本群表现）
- <例："张三在本群是最密集的情绪泄压管道（14 天 N 条里 M 条吐槽），跟 persons/zhang-san.md 档案的'直接且情绪化'特征强化">
- ...

### `## 话题基线`（冷启动时写一版；增量时加/改条目）
- <主题 + 代表内容一两句>

### `## 近期简报`（时间线追加一条）
- `2026-04-22 [冷启动/增量]: <主题 tag 一句话>`

---

## 候选写入：persons/<pinyin>.md

### `persons/zhang-san.md` → `## 互动模式` (新观察)
- "过去 14 天在 X 群表现出'吐槽 + brainstorm 建设性'混合模式，不只是发泄型"

### `persons/li-si.md` → `## 沟通风格` (强化)
- "又一次作为'架构脑'出现——某技术项目是他主动给的架构框架"

---

## 候选更新：checkpoint（groups/<pinyin>.md frontmatter）
- last_processed_at: 2026-04-22T15:04:00+08:00
- last_processed_message_id: om_xxx
```

**关键约束**：
- 每条观察**必须标文件 + 建议 section 归属**（`groups/X.md` → `## Y`，或 `persons/X.md` → `## Y`）——让用户审阅时清楚要落哪
- **section 不是 schema**：如果档案已存在，**优先复用它现有的 section**（通过 Read 拿到现有结构）；如果观察内容跟现有任何 section 都不贴切，建议**新开 section**（起个合适的名字），不要强塞进最接近的那个。新建档案时，起点参考 `templates/` 但 AI 按内容拼合适结构
- 用增量语气：**"新发现 / 强化 / 冲突 / 退潮"** 四种状态，不用"此人/此群是 X"
- **不做"漂亮简报"**——就是候选 diff 清单
- **候选数量按实际信号定，宁缺毋滥**：窗口里没什么新东西就只出 1 条；信号密集就 3-5 条。**不要为了"看起来全面"凑条数**——用户不需要湊数观察，需要真信号。如果窗口全是闲聊没有稳定模式浮现，直接说"这个窗口没有值得沉淀的新观察"，进 Step 6 让用户选 B（只更 checkpoint）或 D（不落）

### Step 6: 询问是否落盘（按 index.md 的通用规则）

**调 `AskUserQuestion` 工具**（不是文本），按 `../shared/memory/index.md` "所有 memory 写入的通用规则"的 4 选项：

- **A. 全部落** —— 按所有候选 diff 写入对应文件
- **B. 只更 checkpoint** —— 忽略观察内容，只 bump `last_processed_at` / `last_processed_message_id`（仅群路径有此选项）
- **C. 让我改一下** —— 用户口头说哪条要删/改/合并 → 重算 diff → 重新展示 → **循环**回到 Step 5 的展示，再调一次 AskUserQuestion
- **D. 不落** —— 本次只看

**如果写入涉及多个文件**（比如同时写 `groups/X.md` 和 `persons/Y.md`），**每个文件单独一个 A/B/C/D 选择**，不要"全部打包 yes/no"——有时用户对群档案满意但人档案还想改。

### Step 7: 按用户选择写入

**写入前再次贴最终 diff**（Edit 或 Write 的完整内容块），然后真执行。

- 新建文件 → Read `../shared/memory/templates/<类型>.md` 拿 schema → Write
- 已有文件 → Edit
- 永远不改 `templates/` 自身

### Step 8: 收尾

```
"还要看别的群/人吗？或者你觉得观察还没写完，可以说 '再加一条 xxx' 我补进去。"
  → 用户要回某条消息 → "用 reply-coach"
  → 用户要手动添加额外观察 → 留在 comm-brief，append 到 diff 再走一次 ABCD
  → 用户要改已有档案的语义画像而不是从消息推 → "用 comm-memory"
```

---

## 共享资源

- Lark CLI 命令：`../shared/references/lark-cli-cookbook.md`
- Memory 权限 / 通用写入规则：`../shared/memory/index.md`
- Memory 目录（读 + 按规则写）：`../shared/memory/`

## 关键约束

- **产物是 diff 候选，不是简报文章。** 展示出来的每一条要能精确映射到 `path → section`
- **写 memory 必须 AskUserQuestion**，按 `index.md` 通用规则 ABCD（含 C 编辑循环）
- **多文件写入分别确认。** 一次 AskUserQuestion 一个文件，别打包
- **别名 / 昵称在群内识别后挂到主档案**——只有**飞书账号真实存在的人**才写 `persons/*.md`（用 `lark_user_id` 做主键，frontmatter 里列别名）；口头提到但无飞书账号的外部人写到 `groups/<群>.md` 的"话题基线"里
- **语气用增量词**：新发现 / 强化 / 冲突 / 退潮，不用"此人是 X"
- **不分析不建议**：触到"这条该怎么回"就说"用 reply-coach"
