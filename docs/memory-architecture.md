# Memory 架构：双层设计

> 设计文档 v0.1 | 2026-04-23

本文件是 Memory 层的权威设计来源，配合 `anti-olden-project-doc.md` §4（技术架构）阅读。代码改动（skill / cookbook / CLAUDE.md）以此为准；偏离需先更新本文件。

---

## 1. 动机

### 1.1 单层架构的问题

Phase 0 当前的 Memory 是**单层**——只落"提炼后的观察"到 `skills/shared/memory/`，原始消息 / 会议转录一次性拉取、分析后丢弃。这种形态在以下四类场景都吃亏：

| 场景 | 现状问题 |
|---|---|
| 同窗口二次分析 | 换一个角度再看某群/某人，整段消息要重拉一遍，API 调用 + token 成本重复花 |
| 纵向回溯 | "过去半年他的语气怎么变的"、"这个群主题迁移的时间线"——memory 里只有当前提炼，看不到原始痕迹 |
| Ad-hoc 查询 | "上周 X 提到 memU 那条消息在哪"——memory 没存原话，要重新调 `+messages-search` 碰运气 |
| Transcript 回看 | 会议逐字稿当前落 `tmp/transcripts/` 用完即删，用户后来想看某场会议原话就又得重新 `vc +notes`；妙记侧的记录也有可能被删 |

### 1.2 信号：Ethan 在"六六6"群的建议

> "感觉有些聊天记录捞出来之后可以保存在本地，以后在这个群组聊天可以续上之前的内容，避免要做长分析时还得重新捞一下数据。文本内容存在本地占用空间比较少，如果算 token 的话每次或许会比较费一点。"
>
> "近几个月的数据就是宝藏啊，我现在经常让它去读取整个群的整体数据，大概三次刀左右的用量，然后它会存到临时目录，然后再分析。"

两个点抓得很准：**文本数据存储成本远低于每次重拉的 token 成本**；**原始数据本身有独立价值**，不只是 memory 的中间产物。

### 1.3 结论

把 Memory 框架拆成**双层**：
- **Raw 层** — 原始数据缓存（新增），跨会话持久
- **Memory 层** — 提炼后的观察（保留现有），专注结构化画像

这不是简单的缓存优化，是两种**不同语义数据**的明确分家——原始数据支持"回放与探查"，提炼观察支持"画像与策略"。

---

## 2. 架构概览

```
skills/shared/
├── memory/                              # Layer 2: 提炼观察
│   ├── persons/<pinyin>_<ou 后6位>.md    # 人物档案
│   ├── groups/<pinyin>.md                # 群档案
│   ├── user-profile.md                   # 用户画像
│   └── templates/                        # 建档模板
├── raw/                                 # Layer 1: 原始数据（新）— gitignore
│   ├── chats/<chat_id>/                  # 群 + 单聊 统一
│   │   ├── _meta.json                    # chat_id / chat_type / name / members / last_fetched_*
│   │   ├── <YYYY-MM>.ndjson              # 按月分桶，主消息逐行追加
│   │   └── threads/<thread_id>.ndjson    # 每个 thread 独立一文件，子消息按时间追加
│   ├── attachments/                      # 图片 / 文件 / 音频
│   │   └── <file_key>.<ext>
│   └── transcripts/<minute_token>/       # 从 tmp/ 升格为长期档案
│       ├── _meta.json                    # title / meeting_time / participants / duration
│       └── transcript.txt
└── tmp/                                 # 短暂 scratch（保留）
```

### 2.1 三层语义边界

| 层 | 内容 | 生命周期 | 读写者 |
|---|---|---|---|
| `memory/` | **已提炼、自然语言、可直接入 LLM 上下文** | 长期，用户共同维护 | 三个 skill 都读；`comm-brief` / `comm-memory` 写 |
| `raw/` | **结构化原始、按需 grep / Read / 视觉加载** | 长期持久，跨会话复用 | 拉取时写；分析时读；用户一般不直接看 |
| `tmp/` | **本次 session scratch**（比如 paginate 的中间页 JSON） | 用完即删 | skill 自己管 |

**核心判断标准：**
- 如果数据经过 AI 思考、需要人类审阅 → `memory/`
- 如果数据是从飞书侧拉下来的原物 → `raw/`
- 如果数据只是这次分析的中间产物 → `tmp/`

---

## 3. Raw 层设计

### 3.1 目录结构

`raw/chats/<chat_id>/` — 群和单聊统一放在这里。飞书 API 层 DM（一对一私聊）也是一个 `chat_id`（`chat_type == p2p`），没必要分目录。`_meta.json` 里标 `chat_type` 区分。

文件名用飞书返回的 `chat_id`（`oc_xxxxx...`），保证稳定主键、不会因改群名断。

### 3.2 Schema

**`_meta.json` — 每个 chat 一份，覆盖写：**

```json
{
  "chat_id": "oc_c60f677dae450cd5e8c494fbf5baab9f",
  "chat_type": "group",
  "name": "六六6",
  "owner_id": "ou_xxxxxx",
  "member_open_ids": ["ou_aaa", "ou_bbb", "ou_ccc"],
  "last_fetched_at": "2026-04-23T16:05:00+08:00",
  "last_fetched_message_id": "om_xxxxxx",
  "first_fetched_at": "2026-04-10T09:30:00+08:00"
}
```

**`<YYYY-MM>.ndjson` — 按月分桶，每行一条消息，追加写：**

每行保留 `+chat-messages-list` 返回的原始 JSON 形态，不做 schema 转换（简单 + 可溯源）。关键字段：

```jsonc
{
  "message_id": "om_xxxxxx",
  "create_time": "2026-04-23 15:49",
  "sender": {"id": "ou_xxxxx", "name": "Ethan(李钊文)", "id_type": "open_id"},
  "msg_type": "text",            // 或 image / file / audio / merge_forward / ...
  "content": "感觉有些聊天记录...", // text 类型；非 text 看 msg_type 定义
  "reply_to": "om_yyyyyy",        // 可能没有
  "thread_id": "omt_zzzzzz",      // 可能没有
  "mentions": [...],              // 原样保留
  // 消息体的其他原始字段原样保留，不删
}
```

按 **UTC+8 本地时间的月**分桶——`create_time` 的年月部分直接截取（飞书 CLI 默认返回 `YYYY-MM-DD HH:MM` 已经是 UTC+8 文本）。跨月的 boundary 消息各自进各自的桶。

### 3.3 追加协议（Append Protocol）

每次 `+chat-messages-list` 返回消息后，skill 按下面顺序处理：

```
for each message (按 create_time 升序):
  1. parse month = message.create_time[:7]          # e.g. "2026-04"
  2. path = raw/chats/<chat_id>/<month>.ndjson
  3. if message_id 已在 path 里 (通过 _meta 的 last_fetched_message_id 推断，或 grep 去重):
       skip
     else:
       append 一行 JSON
  4. 若 msg_type ∈ {image, file, audio, media}:
       下载附件到 raw/attachments/<file_key>.<ext>（见 §3.4）
  5. 更新 _meta.json 的 last_fetched_at / last_fetched_message_id
```

**去重策略：Phase 0 简单做法**是每次追加前用 `grep -F "\"message_id\":\"om_xxxxxx\""` 扫对应月份文件确认没重复；量大再改用单独的 `<chat_id>/_ids.txt` 索引。

**原子性：Phase 0 不保证原子**。skill 追加中断导致半行脏数据的概率极低（`+chat-messages-list` 返回的一批几十条，追加几秒结束），真遇到用户手动修就行。Phase 1 桌面端走 SQLite 事务。

### 3.4 附件处理

**默认下载所有附件**——这是 Ethan 的建议也是合理默认：图片/文件是上下文的一部分（比如某人甩一张报错截图问"这是啥"），memory 层画像需要完整信号。

**路径约定：** `raw/attachments/<file_key>.<ext>`

- `file_key` 来自飞书消息 content 里的 `img_v3_xxx` / `file_xxx` 标识，全局唯一、可跨消息复用（飞书对同一文件去重）
- `<ext>` 根据 `msg_type` 推断：image → `.jpg`/`.png`，file → 原文件扩展名（API 返回），audio → `.m4a`
- 下载命令：`lark-cli im +messages-resources-download --message-id <om> --file-key <fk> --type <t> --output raw/attachments/<fk>.<ext>`（cookbook "消息上下文补全"已有实例）

**LLM 使用方式：**
- **图片** → skill 分析时用 `Read` 工具直接加载（Claude 视觉多模态）
- **文档/文本** → 按扩展名处理，md/txt 直接 Read；docx/pdf Phase 0 暂不自动解析，留给用户需要时手动或走官方 `lark-docs` skill
- **音频** → Phase 0 不处理（需 ASR 流水线），留 file_key 引用即可

**权限：** `drive:file:download` 和 `docs:document.media:download` 已在默认 scope 集合里。

### 3.5 Thread（话题）缓存

飞书里带 `thread_id` 的消息代表这是一个子话题（在飞书客户端呈现为"话题"面板）。主消息出现在月 ndjson 里，但它的所有回复**不散落在月 ndjson**——飞书 API 把 thread 子消息拎到独立命名空间，要用 `+threads-messages-list --thread <id>` 单独拉。

**独立一个文件一个 thread，避免污染月桶：**

```
raw/chats/<chat_id>/threads/<thread_id>.ndjson
```

- 每行一条 thread 子消息，跟主 ndjson 的消息 schema 一致（`message_id` / `sender` / `create_time` / `content` / ...）
- 按 `create_time` 升序追加
- 文件名用 `thread_id`（`omt_xxx`）——一个 thread 一份，不按月分

**为什么分开文件：**
- 月 ndjson 代表"这个月这个群主线的发言"，thread 是"某个小话题的深入讨论"，混在一起会把"这个月群里发生了什么"这类查询搞糊
- thread 通常短（几条到几十条），放独立文件反而好读——想看某 thread 完整内容就直接 cat

**拉取策略（Lazy + 缓存）：**

Phase 0 不做"见到 thread_id 就拉"的激进缓存——很多 thread 永远不会被分析。改成**按需触发 + 结果落盘**：

```
当下游分析（hydration）对某条主消息的 thread_id 有兴趣时：
  1. 先看 raw/chats/<chat_id>/threads/<thread_id>.ndjson 是否存在
     若存在：读本地；若本地最后一条时间距今 < 5 分钟，直接用
     若本地已有但陈旧：用本地 + 拉 `--start <本地最新 create_time + 1s>` 补齐
  2. 若不存在：lark-cli im +threads-messages-list --thread <id> --sort asc --page-size 500 --format json
     → 写 threads/<thread_id>.ndjson
  3. 同一消息的附件下载规则跟主消息一致（§3.4）
```

**好处：** 第一次分析付出一次 API 调用，之后同 thread 的后续讨论只追加增量；避免重复拉。

### 3.6 Transcript 从 tmp 升格到 raw

会议逐字稿**不再用完即删**。`vc +notes` 输出直接落：

```
raw/transcripts/<minute_token>/
├── _meta.json                  # title / meeting_time / participants / duration / note_doc_token / verbatim_doc_token
└── transcript.txt              # 逐字稿纯文本（与 PR #1 里的格式一致）
```

这是对 PR #1 "用完即删"决策的反转——Ethan 的原则同样适用：**转录文本是宝藏**，存下来的增量存储成本远低于重新调 `+notes` 的 token 成本，且能支持纵向回顾（"X 在三月跟四月的会议里对同一问题的说法一致吗"）。

对应 comm-brief 的改动：`Step 9` 不再 `rm -rf tmp/transcripts/<token>/`；transcript 路径统一指向 `raw/transcripts/<token>/`。

---

## 4. Memory 层（保留 + 明确边界）

现有的 `memory/persons/` / `memory/groups/` / `memory/user-profile.md` **结构不变**。唯一的语义调整是：

### 4.1 Checkpoint 字段从 Memory 迁移到 Raw

原本的 `groups/<group>.md` frontmatter 里有：

```yaml
last_processed_at: 2026-04-22T15:04:00+08:00
last_processed_message_id: om_xxxxxx
```

这些字段本质是"原始数据拉到哪里了"——是 **IO 状态**，不是**画像内容**。迁到 `raw/chats/<chat_id>/_meta.json` 的 `last_fetched_at` / `last_fetched_message_id`。

**好处：** `memory/` 文件专注承载"观察"，用户读档案时不再看到 IO 状态污染。`raw/` 是 IO 操作的真源，两层各司其职。

**迁移：** 用户本地的现有 groups memory 里如果已经有 checkpoint，首次 comm-brief 增量跑时把值拷贝到 `_meta.json` 后再从 memory 文件里清掉；没有就直接从新起。

### 4.2 Memory 还记什么

保持原设计：
- **沟通风格 / 雷区 / 互动模式 / 有效策略 / 关系备注 / 近期简报 / 话题基线 / 关键人物**
- 都是经过 skill + 用户共同审过的提炼
- **每条观察打来源标签**：`[消息]` / `[会议:<标题>]` / 未来其他源
- 不记 IO 状态、不记原始文本（原文只当引文短片段出现在观察里）

---

## 5. 数据流（comm-brief 新流程）

### 5.1 群路径

```
用户请求"看 xx 群最近啥情况"
  ↓
Step 1: 确认范围
  ↓
Step 2: 读 raw/chats/<chat_id>/_meta.json
  ↓ (未命中 → 冷启动；命中 → last_fetched_at + 1s 起增量)
Step 3: lark-cli im +chat-messages-list --start <since> …
  → 按追加协议写 raw/chats/<chat_id>/<month>.ndjson
  → 附件下载到 raw/attachments/
  → 每条带 thread_id 的主消息：按 §3.5 的 lazy 规则拉 thread 子消息到 threads/
  → bump _meta.json 的 last_fetched_*
  ↓
Step 4: 读 memory/groups/<pinyin>.md（如有）+ memory/user-profile.md + 相关 persons/
  ↓
Step 5: 从 raw 拿出本次窗口的消息（主 + thread 子消息扁平化）→ 按 analyze.md 生成候选观察
  ↓ (带 [消息] 来源标签)
Step 6-8: 口述补充 / ABCD 确认 / 写入 memory
  ↓
Step 9: 收尾（不删 raw；不删 transcript 缓存）
```

### 5.2 人物路径（含会议转录）

跟群路径类似，但 Step 3 会分两支：

```
Step 3a (消息支): 对所有共同群 + DM 跑 +chat-messages-list，追加到 raw/chats/
Step 3b (会议支, 若选了): vc +search → +notes --output-dir raw/transcripts/<token>/
Step 4-5: 从 raw 读聚合上下文 → 按 analyze.md + analyze-transcript.md 分别跑 → 合并 diff
Step 6-9: 同上
```

**关键变化：** 多次分析同一人时，raw 已有的数据不用重拉；只追加窗口外的新消息和尚未缓存的会议。

### 5.3 reply-coach / comm-memory 的配套

- **reply-coach** 读 raw 拿回复上下文（之前是每次重拉）——降低起草一次回复的 token / 延迟
- **comm-memory** 是纯口述流，不涉及 raw——维持原样

---

## 6. 分桶、去重、GC

### 6.1 分桶粒度

`<YYYY-MM>.ndjson` 按月。理由：
- 活跃群单月 ndjson 大致 1-10MB 量级，grep + jq 在 Mac 上毫秒级
- 查询常按月窗口组织（"过去 14 天" / "过去一个月" / "Q1" 自然落在月边界附近）
- 年分桶太粗（单文件几十 MB 时 grep 慢）；日分桶太碎（1000+ 文件的 chat 目录读起来累）

### 6.2 去重

消息以 `message_id` 为主键。追加前检查是否已在对应月 ndjson 出现。Phase 0 用 `grep`，Phase 1 桌面端上 SQLite 后改主键约束。

### 6.3 GC 策略

**Phase 0 不做。** 原因：
- 文本数据增长慢，活跃群半年也就百 MB 量级，1-2 GB 前完全不用担心
- 不知道用户最终会怎么用 raw（纵向分析？原文 grep？），过早 GC 反而破坏场景
- 用户本地盘空间通常 >= 256GB，这点数据不值得维护 GC 状态机

**达到 GC 触发条件前的处置：** 本节只记录决策，不生成代码。到超过 1-2 GB 时考虑：
- 按 chat_id + 月保留窗口（比如每群最多近 12 个月，更老的归档到 `raw/archive/<chat_id>.tar.gz`）
- 或按 chat_id 最后 fetch 时间做 LRU（半年没打开的群归档）
- attachments 单独 GC（按 file_key 最后引用时间）

---

## 7. Phase 0 ↔ Phase 1 的映射

| 维度 | Phase 0（Skill） | Phase 1（桌面端 Tauri v2） |
|---|---|---|
| Raw 存储 | `skills/shared/raw/` 下的 ndjson + json | SQLite 表：`messages`、`chats`、`attachments`、`transcripts` |
| Memory 存储 | `skills/shared/memory/` 下的 markdown | SQLite 表 + markdown 导出（用户可直接编辑） |
| 附件 | 本地文件 `raw/attachments/<file_key>.<ext>` | 同上，但由 SQLite 维护引用计数 |
| 去重 | `grep` 消息 ID | SQLite PRIMARY KEY |
| 事务 | 无（追加失败用户手修） | SQLite 事务 |
| 向量检索 | 不做 | ChromaDB 索引所有 ndjson + memory |
| 多用户 | 每人一份私仓 clone（memory 与 raw 都 gitignore） | 每台机器一份 SQLite（见 `project_memory_is_per_user.md`） |

**迁移路径：** Phase 1 的 migration 工具会把 Phase 0 的 ndjson 按行读入 SQLite，schema 按当前 JSON 字段对齐。所以现在 ndjson 保留原始 API 响应结构 = 未来迁移最省事。

---

## 8. 权限与隐私

### 8.1 所需 scope（变化）

Raw 层本身不需要新 scope——读消息靠的还是 `im:message:readonly` / `im:message.group_msg:get_as_user` / `im:message.p2p_msg:get_as_user`。附件下载走 `drive:file:download` / `docs:document.media:download`，都已在当前 16 条最小集内。

### 8.2 多用户隔离

`raw/` 跟 `memory/` 一样 **gitignore + 每人本地**，不合并不同步（见 `project_memory_is_per_user.md`）。三人并发使用时各自维护自己的原始缓存，不存在数据串扰。

### 8.3 敏感信息

raw ndjson 里是原封的飞书消息——包含真实姓名 / 真实聊天内容 / 可能的敏感截图。**绝不上传到仓库、绝不分享给协作者、绝不作为 AI 训练数据导出**。符合 CLAUDE.md 里"Skill 文件绝不写真实姓名/ID"的约束（skill 源码不含，但 raw 允许——就像 memory 允许一样）。

---

## 9. 开放问题

1. **删除同步问题。** 用户在飞书里撤回消息、管理员清理群——raw 里不会感知。长期会有"raw 比飞书侧多"的漂移。Phase 0 接受这个漂移（自用工具，漂移是正向——撤回的消息我们也想看到原话）。如果产品化分发要考虑。

2. **Reply-coach 复用 raw 的确切时机。** 现在每次起草都重拉近 20 条。改成"优先 raw + 补拉最近 N 条"需要评估：reply-coach 对"最新消息"要求更高，缓存可能差几秒就过时。方案：reply-coach 仍然每次实拉，**但把结果也写进 raw**（相当于免费白给 comm-brief 省一次拉取）。

3. **超大群冷启动。** 陌生群 14 天冷启动可能数千条消息，单次 `paginate` 脚本可能跑几十页。raw 层让这个问题显性化（一次拉完就缓存住），但体验上还是卡一阵子。Phase 0 可接受，Phase 1 可以做渐进式拉取（后台补齐）。

4. **Thread 的"chat 归属"重命名**。thread 文件放在 `raw/chats/<chat_id>/threads/` 下——如果一个 thread 被转发到另一个群继续讨论（飞书支持），理论上同一 `thread_id` 可能在两个 chat 目录下都需要。Phase 0 先不处理（罕见），真遇到再设计 symlink 或集中式 `raw/threads/` 路径方案。

---

## 10. 变更记录

| 版本 | 日期 | 变更 |
|---|---|---|
| v0.1 | 2026-04-23 | 初稿。采纳 Ethan 关于"原始数据缓存有长期价值"的建议；将 Phase 0 的 memory 架构从单层扩为双层 |
