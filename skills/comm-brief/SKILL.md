---
name: comm-brief
description: 重流程——从飞书消息和会议转录出发给群/人做 memory 分析：拉 lark-cli 消息/转录 → AI 分析出候选观察 → 主动问用户"你有啥要一起加的认知" → 合并口述和外部观察 → 用户审+改+落。触发：用户只给了对象没给描述，比如"给 X 建档"/"xx 群最近啥情况"/"给 xx 群建档"/"最近半个月 xx 群有什么变化"/"看跟 X 最近互动"/"今天谁 @ 我了"/"从最近那场会看看 X 的表现"。如果用户已经把完整口述讲出来了（"X 是 xxx 风格，雷区是 xxx"），走 comm-memory 而不是本 skill。
allowed-tools: Bash Read Write Edit AskUserQuestion
---

# 沟通简报（消息驱动的 memory 沉淀）

## 你要做什么

**从消息里观察 → 生成候选 memory 更新 → 让用户审阅+改+落盘。**

不是"给用户看漂亮简报"——展示出来的内容本身就是**拟写入 memory 的 diff**。用户看的是"这些观察要不要落档"，不是"总结文章"。

两种输入形态：
- **群动态**（某个 chat_id + 时间窗）→ 冷启动 14 天 / 增量按 checkpoint
- **人物互动**（某个 ou_id + 时间窗）→ 聚焦该人近期表现，数据源可选"消息 / 消息+会议转录 / 只会议转录"

会议转录是人物路径专有的补充源——群里的人是他愿意让你看的形象，会议里才漏尾巴（插话、hedging、被追问时的反应等）。群路径不走转录。

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
- 跟某人的近期互动 / 给某人建档 → 再问"是哪个人？"

如果用户已经指定了群名，用 `+chat-search --query "<关键词>"` 消歧后**调 AskUserQuestion 让用户确认**"是 `oc_xxx` 这个群吧"。

**人物路径必问：代号/别名**。同事群里经常用代号称呼（例：数字代号 / 表情符号代号 / 英文名缩写 / 外号——都替代真名出现），**光搜真名会漏一半信号**。人物建档时一定要调 `AskUserQuestion` 问一次：
> "你们群里叫他有别的称呼吗？比如数字代号、绰号、英文名、外号？"

用户回答的代号要加进后续 Step 3 / Step 4 的文本过滤关键词。

**人物路径必问：数据源**。再调一次 `AskUserQuestion` 让用户选画像的输入：
> "分析数据源：消息（群+DM）/ 消息 + 会议转录 / 只看会议转录？"

- **消息** — 默认、最快，适合大多数人
- **消息 + 会议转录** — 用户要"完整画像"、或关心这人"线下尾巴"时
- **只看会议转录** — 用户明确说"从 X 场会看看他"时

带"会议转录"的分支会走 Step 3B。

### Step 2: 读 raw/_meta + memory 档案 → 决定窗口

**拉取状态（checkpoint）归 raw，画像归 memory。** 两处分别读：

**群路径**：
- **IO 状态** → `../shared/raw/chats/<chat_id>/_meta.json`
  - 命中：读 `last_fetched_at` / `last_fetched_message_id` 作为增量锚点
  - 未命中：冷启动（首次见到该群）
- **画像** → glob `../shared/memory/groups/*.md`，frontmatter `chat_id` 匹配 → Read（拿"话题基线" / "近期简报" / 已有观察）
  - 画像文件不存在不报错；可能已拉过消息但还没写过画像

**窗口判断**：

| 条件 | 模式 | 拉取窗口 |
|---|---|---|
| `_meta.json` 不存在 / `last_fetched_at` 距今 > 30 天 | **冷启动** | 默认最近 **14 天** |
| `last_fetched_at` 在 30 天内 | **增量** | `--start = last_fetched_at + 1s` |
| 用户显式指定 | **自定义** | 按用户 |

**告诉用户选了哪个模式**（文本说明，不用 AskUserQuestion）：
> "这个群 10 天前处理过，走增量模式，拉 2026-04-12T... 起的消息。"

**人物路径**：
- 画像：glob `persons/*.md`，frontmatter `lark_user_id` 匹配
- IO 状态：按目标参与的每个**共同 chat**，分别读其 `raw/chats/<chat_id>/_meta.json`——人物路径是多 chat 聚合，所以 checkpoint 是 per-chat 的

### Step 3: 拉消息 → 写 raw（按追加协议）

**所有拉取都要落盘 raw**。查 `../shared/references/lark-cli-cookbook.md` 的 **"Raw 数据缓存落盘协议"** 章节——脚本实现直接复制，别自己重写。核心三件事：
1. 追加主消息到 `raw/chats/<chat_id>/<YYYY-MM>.ndjson`（按 create_time 月分桶，去重靠 message_id）
2. 每条 `msg_type ∈ {image, file, audio, media}` 的消息**默认下载附件**到 `raw/attachments/<file_key>.<ext>`
3. 每条带 `thread_id` 的主消息，按 lazy 规则拉 thread 到 `raw/chats/<chat_id>/threads/<thread_id>.ndjson`
4. bump `_meta.json` 的 `last_fetched_at` / `last_fetched_message_id`

**命令：**
- **群动态**：`+chat-messages-list --chat-id <id> --start <ISO8601> --sort asc`（注意用 asc 方便月分桶顺序追加），走分页脚手架直到 `has_more=false` 或跨出窗口
- **人物互动**：对每个共同 chat 走群动态流程；私聊（DM）同样走 `+chat-messages-list --user-id <ou>` 并按 p2p chat_id 落 raw
- **今日 @**：cookbook "仍需 Phase 0 发现"段；短期兜底 = 逐 chat 拉近 24h + `jq` 过滤 `mentions` 含 user `userOpenId`。无论哪种路径，**经过的消息都要写 raw**——@ 也是群消息的一部分

**检查点：拉完 raw 先做一次完整性校验**（避免半拉状态误判下次增量）：
- `_meta.last_fetched_message_id` 应该能 `grep` 到对应月桶里
- 附件全部存在（for each file_key in raw messages → check raw/attachments/）

**Hydration 与 raw 的关系：** 现在 hydration（thread / reply_to / 附件）其实就是"走追加协议"的一部分——缓存已经做了。分析时 reply_to 指针直接在月 ndjson 里找；找不到再 `+messages-mget` 拉补回 raw（补到原月桶）。

### Step 3B: 拉会议转录（仅人物路径，Step 1 选了含"会议转录"时）

查 `../shared/references/lark-cli-cookbook.md` 的 "会议转录拉取" + "Raw 数据缓存落盘协议 → Transcript"两段。核心三步：

1. **搜会议**：`lark-cli vc +search --participant-ids <ou_target> --start <window> --end <now>` → 列出目标参与过的会议（title / minute_token / start_time / duration）
2. **用户选场次**：调 `AskUserQuestion` 让用户勾选要分析哪几场——不要全量拉（每场 transcript 动辄数万字，整窗口拉会爆上下文，且很多会议只是例会/顺口提）。默认建议用户挑 1-3 场"议题密集"的
3. **下载到 raw**：对每个选中 `minute_token` 跑 `lark-cli vc +notes --minute-tokens <token> --output-dir ../shared/raw/transcripts/<token>/ --format json > ../shared/raw/transcripts/<token>/_meta.json` → transcript.txt + _meta.json 落 raw，**跨会话持久**
4. **本地优先**：如果 `raw/transcripts/<token>/transcript.txt` 已存在就跳过下载——妙记逐字稿一旦生成不会变，缓存值得信任

**说话人 → open_id 归属**：transcript 里的说话人标签是**显示名**（`EnglishName(中文名)`），不是 `open_id`。写观察前用 `contact +search-user --query "<中文名>"` 确认解析结果的 open_id 跟目标一致——同名 / 音近名的误伤会直接污染档案。

**文件生命周期**：raw/transcripts/ 已 gitignore，**跨会话持久不删**。同一会议后续再分析直接读本地，不再调 API。

### Step 4: 读其他 memory（背景上下文）

- Read `user-profile.md`（知道"我"是谁 → 影响"我"在本群的观察角度）
- 涉及的群内成员 → glob `persons/*.md` + frontmatter `lark_user_id` 匹配 → Read（看"过去档案"是什么样，好判断"这次新观察是强化还是冲突"）
- 涉及的群 → group 档案已在 Step 2 Read 过

**Memory 不存在不报错、不中断。**

### Step 5: 生成**变化观察清单**（diff-shape，不是状态快照）

**核心心法**：每条观察都要能直接映射到某个 memory 文件的某个 section。用**"新发现 / 强化 / 冲突 / 退潮"**这种**增量语气**，不用"这个群是什么样"的描述语气。

**按源分别跑分析 prompt：**
- 消息 → `../shared/prompts/analyze.md`
- 会议转录 → `../shared/prompts/analyze-transcript.md`（含"线下尾巴"观察框架）

两者产出的观察合并进同一份 diff 清单，但**每条观察必须标明来源**：`[消息]` 或 `[会议:<会议标题>]`——这样用户审阅时知道这条信号是从群聊还是会议里来的。

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
- [消息] "过去 14 天在 X 群表现出'吐槽 + brainstorm 建设性'混合模式，不只是发泄型"

### `persons/li-si.md` → `## 沟通风格` (强化)
- [消息] "又一次作为'架构脑'出现——某技术项目是他主动给的架构框架"
- [会议:Plaud Kickoff] "00:14:32 他对细节被追问时直接说'这个我没想过'——跟消息里'深思熟虑'的印象有微冲突，疑似会议里更坦诚"
```

**注意：不再需要"候选更新：checkpoint"段**——checkpoint 已经在 Step 3 拉消息时自动 bump 到 `raw/chats/<id>/_meta.json`，跟用户的 memory 写入决策解耦。用户 ABC 选 C（不写 memory）也不影响下次增量起点。

**关键约束**：
- 每条观察**必须标文件 + 建议 section 归属**（`groups/X.md` → `## Y`，或 `persons/X.md` → `## Y`）——让用户审阅时清楚要落哪
- **section 不是 schema**：如果档案已存在，**优先复用它现有的 section**（通过 Read 拿到现有结构）；如果观察内容跟现有任何 section 都不贴切，建议**新开 section**（起个合适的名字），不要强塞进最接近的那个。新建档案时，起点参考 `templates/` 但 AI 按内容拼合适结构
- 用增量语气：**"新发现 / 强化 / 冲突 / 退潮"** 四种状态，不用"此人/此群是 X"
- **不做"漂亮简报"**——就是候选 diff 清单
- **候选数量按实际信号定，宁缺毋滥**：窗口里没什么新东西就只出 1 条；信号密集就 3-5 条。**不要为了"看起来全面"凑条数**——用户不需要湊数观察，需要真信号。如果窗口全是闲聊没有稳定模式浮现，直接说"这个窗口没有值得沉淀的新观察"，进 Step 7 让用户选 B（只更 checkpoint）或 D（不落）

### Step 6: 主动问用户口述补充（**不要跳**）

**展示 draft 后，不要直接进 Step 7 的写入确认**——先给用户机会把 AI 看不到的认知加进来。

调 `AskUserQuestion` 工具，问：

> "我基于消息的观察如上。你对 {这个人 / 这个群} 还有什么认知要一起加进去的吗？比如：我看不到的背景、过往合作史、雷区、汇报线、组织层信息……"

选项：
- **A. 我补一下** —— 用户在 "Other" 自由输入框写补充内容 → AI 把口述**融合进 draft**（明确标注"来自用户口述"跟"来自消息观察"分开）→ 重新展示合并后的 draft → **再问一次** A/B（循环直到用户选 B 或说"够了"）
- **B. 不用，就按 draft 走** —— 直接进 Step 7

**为什么必须这一步**：消息只是外部观察，用户脑子里的认知往往更关键（比如"他是我老板的儿子"、"上次开会他吐槽过 X"）——如果 AI 只拿消息产物去问 Step 7 ABCD，用户得在"C. 让我改"里临时挤出所有认知，信息丢失率高。Step 6 的"主动问"是让用户从容地把认知倒出来。

### Step 7: 询问是否落盘（按 index.md 的通用规则）

**调 `AskUserQuestion` 工具**（不是文本），按 `../shared/memory/index.md` "所有 memory 写入的通用规则"的 3 选项：

- **A. 落** —— 按候选 diff 写入对应文件
- **B. 让我改一下** —— 用户口头说哪条要删/改/合并 → 重算 diff → 重新展示 → **循环**回到 Step 5 的展示，再调一次 AskUserQuestion
- **C. 不落** —— 本次只看

IO 状态（`last_fetched_*`）跟用户选 A/B/C 无关——Step 3 拉消息时已经自动 bump 到 raw/_meta.json。

**如果写入涉及多个文件**（比如同时写 `groups/X.md` 和 `persons/Y.md`），**每个文件单独一个 A/B/C 选择**，不要"全部打包 yes/no"——有时用户对群档案满意但人档案还想改。

### Step 8: 按用户选择写入

**写入前再次贴最终 diff**（Edit 或 Write 的完整内容块），然后真执行。

- 新建文件 → Read `../shared/memory/templates/<类型>.md` 拿 schema → Write
- 已有文件 → Edit
- 永远不改 `templates/` 自身

### Step 9: 收尾

**不再删 raw**（消息 / transcript / 附件都是长期档案）。本步只做语义收尾：

```
"还要看别的群/人吗？或者你觉得观察还没写完，可以说 '再加一条 xxx' 我补进去。"
  → 用户要回某条消息 → "用 reply-coach"
  → 用户要手动添加额外观察 → 留在 comm-brief，append 到 diff 再走一次 ABC
  → 用户要改已有档案的语义画像而不是从消息推 → "用 comm-memory"
```

**唯一的清理**是 `tmp/` 下本 session 自己创建的中间文件（比如分页脚手架的 `/tmp/paginate-out/*.json`）——但这些是 OS 临时目录下的文件，OS 自己会清。

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
