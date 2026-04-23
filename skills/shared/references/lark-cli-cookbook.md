# Lark CLI Cookbook

本文件是 Skill 运行时**查命令的首选来源**。cookbook 里没有的，用 `lark-cli schema <path>` 或 `lark-cli <cmd> --help` 发现后再写，**不要凭记忆猜语法**。

Lark CLI：https://github.com/larksuite/cli
**二进制名：`lark-cli`**（不是 `lark`——Skill 里统一用 `lark-cli`）

---

## 前置准备

### 安装
```bash
npm install -g @larksuite/cli
```

### 首次配置（一次性）
```bash
lark-cli config init --new          # 配置应用凭证
lark-cli auth login --recommend     # OAuth 浏览器授权，自动推荐常用权限
lark-cli auth status                # 验证登录
```

若需更细粒度的 scope：
```bash
lark-cli auth login --domain calendar,task
lark-cli auth login --scope "im:message.group_msg"
lark-cli auth scopes                # 列出可用 scope
lark-cli auth check --scope "<scope>"  # 验证当前 token 是否有某个权限
```

### 登录状态
```bash
lark-cli auth status     # 当前登录身份
lark-cli auth list       # 所有已登录账号
lark-cli auth logout     # 登出
```

### 查当前用户身份（"我是谁"）

运行时拿当前登录用户的 `lark_user_id`（open_id），用于过滤 "@ 我" / "我发的消息"：
```bash
lark-cli auth status --format json | jq -r '.userOpenId'
# 也可以一次拿全：jq '{name:.userName, open_id:.userOpenId, app_id:.appId}'
```
**不要把 ou_id 持久化到 memory markdown 里。** memory 正文只写自然语言描述，ID 运行时查 CLI。

### 身份切换（用户 vs 机器人）
不用 subcommand 切换，而是在命令上加 `--as`：
```bash
lark-cli im +messages-send --as user --chat-id "oc_xxx" --text "hi"
lark-cli im +messages-send --as bot  --chat-id "oc_xxx" --text "hi"
```

---

## 命令结构与发现

Lark CLI 有**两种命令形态**，同时存在：

### A. Shortcut 命令（`+` 前缀）
高频操作的简写，参数直接给：
```bash
lark-cli im +messages-send --chat-id "oc_xxx" --text "Hello"
lark-cli docs +create --title "Weekly" --markdown "# Progress"
lark-cli calendar +agenda
```

### B. 原始 API 调用（覆盖全部 2000+ 飞书 Open API 端点）
```bash
lark-cli api GET /open-apis/<path>            --params '{"k":"v"}'
lark-cli api POST /open-apis/<path>           --params '{...}' --data '{...}'
```

### 发现命令的顺序（**按此顺序查**）

1. `lark-cli <service> --help` — 该服务下有哪些 shortcut 和子命令
2. `lark-cli schema` — 列出所有 schema 节点
3. `lark-cli schema <service>.<resource>.<action>` — 看某个 API 的**完整参数和类型**（如 `lark-cli schema im.messages.list`）
4. 参考飞书 Open API 官方文档 REST 路径，用 `lark-cli api <METHOD> /open-apis/...` 兜底

---

## 输出格式（`--format`）

| 值 | 适用 |
|---|---|
| `json` | 程序解析用——**Skill 默认选这个**，下游 `jq` 处理 |
| `ndjson` | 管道处理大量行 |
| `pretty` | 人类阅读的完整 JSON |
| `table` | 简短表格总览 |
| `csv` | 导出 |

---

## Dry-run（写操作前预演）

写操作加 `--dry-run` 查看将要发送的 payload，不真执行：
```bash
lark-cli im +messages-send --chat-id "oc_xxx" --text "hi" --dry-run
```

**Skill 在向用户确认发送前，先跑一次 `--dry-run` 把完整 payload 贴出来给用户看，再真发。**

---

## 常用配方

下面的 shortcut 命令名来自 Lark CLI 官方 README 和 `lark-im` skill 文档（shortcut 名已确认），**但每个 shortcut 的 flag 细节**首次使用时要跑 `lark-cli <cmd> --help` 或 `lark-cli schema <path>` 验证，然后把完整语法覆盖回本行。

### 消息读取

**拉某 chat 的历史消息**（支持时间范围、排序、分页）
```bash
lark-cli im +chat-messages-list --chat-id "<oc_xxx>" \
  --page-size 50 --sort desc --format json
# 可选：--start / --end（ISO 8601） / --page-token
# 也可用 --user-id "<ou_xxx>" 拉 P2P（与 --chat-id 互斥）
```

**跨聊天搜消息**（关键词 / sender / 时间范围）
```bash
lark-cli im +messages-search --keyword "<关键词>" --format json
# 可选：--sender <user_id>, --chat-id <chat_id>
# 首次用：lark-cli im +messages-search --help（flag 名未实测）
```

**批量取多条消息详情**
```bash
lark-cli im +messages-mget --message-ids "om_xxx,om_yyy" --format json
# 上限 50 个 message_id；自动做 sender 名字补全；自动展开 thread replies
```

**获取单条消息详情**（若 mget 单条不适合，走 raw API）
```bash
lark-cli api GET /open-apis/im/v1/messages/<message_id> --format json
```

### 分页拉取脚手架（窗口内自动翻到底）

活跃群半年窗口可能几千条，50 条/页要翻 30-50 页。写成可复用脚本：

```bash
cat > /tmp/paginate.sh <<'EOF'
#!/bin/bash
# Usage: paginate.sh <chat_id> <label> <start_iso8601> [page_cap]
CHAT_ID="$1"; LABEL="$2"; START="$3"; CAP="${4:-200}"
OUTDIR="/tmp/paginate-out"; mkdir -p "$OUTDIR"
TOKEN=""; PAGE=1
while :; do
  if [ -z "$TOKEN" ]; then
    lark-cli im +chat-messages-list --chat-id "$CHAT_ID" --start "$START" --page-size 50 --sort desc --format json > "$OUTDIR/${LABEL}_page${PAGE}.json"
  else
    lark-cli im +chat-messages-list --chat-id "$CHAT_ID" --start "$START" --page-size 50 --sort desc --page-token "$TOKEN" --format json > "$OUTDIR/${LABEL}_page${PAGE}.json"
  fi
  HAS_MORE=$(jq -r '.data.has_more' "$OUTDIR/${LABEL}_page${PAGE}.json")
  TOKEN=$(jq -r '.data.page_token // ""' "$OUTDIR/${LABEL}_page${PAGE}.json")
  EARLIEST=$(jq -r '.data.messages[-1].create_time // "∅"' "$OUTDIR/${LABEL}_page${PAGE}.json")
  echo "[$LABEL] page$PAGE earliest=$EARLIEST has_more=$HAS_MORE"
  if [ "$HAS_MORE" != "true" ] || [ -z "$TOKEN" ] || [ "$TOKEN" = "null" ]; then break; fi
  PAGE=$((PAGE+1))
  if [ $PAGE -gt $CAP ]; then echo "[$LABEL] CAP HIT at $CAP"; break; fi
done
echo "[$LABEL] DONE. pages=$PAGE"
EOF
chmod +x /tmp/paginate.sh
# 用法：/tmp/paginate.sh "oc_xxx" "mygroup" "2025-11-01T00:00:00+08:00" 100
```

**`page_cap` 怎么设**（踩过的坑）：
1. **先 peek 第一页**看 50 条的时间跨度——`latest` 减 `earliest` = 单页覆盖多少小时
2. **估算**：目标窗口总时长 / 单页跨度 = 需要的页数，再 × 1.5 余量
3. **活跃群半年可能要 50-200 页**。默认 50 cap 对话唠群是不够的（实测：某高活跃闲聊群 50 页只覆盖 2 个月）
4. 后台跑：用 `run_in_background=true` 起多个群并行，避免串行等

### 消息上下文补全（hydration）

消息上下文常常**跨窗口**——仅凭 `+chat-messages-list` 返回的最近 N 条不够，需要额外拉 3 种东西才能还原完整语境。分析前必须 hydrate，不要在半残信息上做推断。

**什么时候要 hydrate：**
| 触发条件 | 要拉什么 | 用哪个命令 |
|---|---|---|
| 消息 `reply_to` 不为空、且目标 `om_xxx` 不在当前窗口 | 拉这条被回复的原消息（递归 follow 直到链首） | `+messages-mget` |
| 消息带 `thread_id` 或 `msg_type: merge_forward` | 拉整个 thread 的所有消息 | `+threads-messages-list` |
| `msg_type: image / file / audio / media` 且上下文需要"对方发了什么" | 下载资源到本地 → 让 Claude 直接 Read（图片走视觉，文件按扩展名处理） | `+messages-resources-download` |

**#1 还原 reply_to 链**
```bash
# 目标：对消息 M 做分析前，把 M.reply_to → P, P.reply_to → PP, ... 全拉出来
lark-cli im +messages-mget --message-ids "<om_parent>[,<om_grandparent>,...]" --format json
# 策略：批量一次最多 50 个 id；递归 follow 时注意去重、记录链序
```

**#2 展开 thread**
```bash
# --thread 接 om_xxx 或 omt_xxx 都行，CLI 会解析成 thread_id
lark-cli im +threads-messages-list --thread "<om_xxx 或 omt_xxx>" \
  --sort asc --page-size 500 --format json
```

**#3 下载图片/文件**
```bash
# content 字段里的占位符 "[Image: img_v3_xxx]" 里那串就是 file-key
lark-cli im +messages-resources-download \
  --message-id "<om_xxx>" \
  --file-key "<img_v3_xxx 或 file_xxx>" \
  --type image \
  --output "/tmp/anti-olden/<om_xxx>-<key>.jpg"
# --output 限相对路径且不能有 .. ——但可以给 /tmp 下的绝对路径吗？实测时验证
# 下载后：图片用 Read 工具直接加载（Claude 有视觉），文件按扩展名处理
```

**实战顺序（reply-coach Step 2 流程）：**
1. `+chat-messages-list` 拉近 20-30 条
2. 扫一遍：找出有 `reply_to` 但父消息不在窗口的 → 收集 `om_id`
3. 找出有 `thread_id` 的 → 标记
4. 找出 `msg_type != text` 的媒体消息 → 标记
5. 一次 `+messages-mget` 批量拉 reply 父消息；对每个 thread 调 `+threads-messages-list`；对目标上下文里的媒体调 `+messages-resources-download`
6. 把 hydrate 后的完整消息树扁平化成时间轴喂给分析 prompt

### 消息写入

**所有写操作的流程：** 先 `--dry-run` 把 payload 贴给用户看 → 用户说"发"后去掉 `--dry-run` 真跑。

**发送文本消息**
```bash
lark-cli im +messages-send --chat-id "<oc_xxx>" --text "<text>" --dry-run
# 可选：--as user|bot 切身份
```

**回复某条消息形成 thread**
```bash
lark-cli im +messages-reply --message-id "<mid>" --text "<text>" --dry-run
```

### 群与联系人

**按群名搜群（找 chat_id）**
```bash
lark-cli im +chat-search --query "<群名关键字>" --format json
# ⚠️ flag 是 --query 不是 --keyword（实测 2026-04-22）。群名模糊匹配靠得住，短词比长词好；注意**空格/连字符敏感**——群名若含空格，搜索词带连字符可能搜不到，先试空格版本
```

**找"我跟某人都在的群"**（人物建档必需）
```bash
lark-cli im +chat-search --member-ids "<ou_target>,<ou_me>" --format json \
  | jq '.data.chats[] | {chat_id, name}'
# 返回我和 ou_target 共同在场的所有群
```

**按姓名搜用户拿 ou_id**
```bash
lark-cli contact +search-user --query "<中文名或英文名>" --format json \
  | jq '.data.users[] | {open_id, name, en_name, department_ids}'
```

**联系人操作** — 有专门的 `lark-contact` skill。首次用时查：
```bash
lark-cli contact --help
lark-cli schema contact.users
```

**反查 user_id（按 email/mobile）**（原始 API，可能有更简 shortcut）
```bash
lark-cli api POST /open-apis/contact/v3/users/batch_get_id \
  --params '{"user_id_type":"open_id"}' \
  --data '{"emails":["xxx@company.com"]}'
```

**拿用户详情（已知 user_id）**
```bash
lark-cli api GET /open-apis/contact/v3/users/<user_id> \
  --params '{"user_id_type":"open_id"}' \
  --format json
```

### 人物建档：跨群 + DM 聚合（comm-brief 人物路径）

给某人建档时，要把"**我跟他所有共同语境**"聚合起来分析。完整流程：

```bash
# === Step 0: 拿 ID ===
MY_OU=$(lark-cli auth status --format json | jq -r '.userOpenId')
TARGET_OU=$(lark-cli contact +search-user --query "<姓名>" --format json | jq -r '.data.users[0].open_id')
echo "me=$MY_OU target=$TARGET_OU"

# === Step 1: 找共同群 ===
lark-cli im +chat-search --member-ids "$TARGET_OU,$MY_OU" --format json \
  | jq -r '.data.chats[] | "\(.chat_id) \(.name)"'
# → 输出共同群列表，跟用户确认挑哪几个群作为分析范围

# === Step 2: 每个共同群分页拉（起 background，并行）===
# 用 /tmp/paginate.sh <chat_id> <label> "2025-11-01T00:00:00+08:00" 200
# 起多个 background 任务并行；活跃群设 cap=100-200

# === Step 3: 拉私聊（DM）===
lark-cli im +chat-messages-list --user-id "$TARGET_OU" --start "2025-11-01T00:00:00+08:00" \
  --page-size 50 --sort desc --format json > /tmp/paginate-out/dm.json
# DM 一般量小，一次拉完

# === Step 4: 聚合 + 提取冲突信号 ===
# 扁平化：所有群所有页合并 + 按时间排序 + 带 mid / reply 指针
jq -s '[.[].data.messages[] | {t: .create_time, n: .sender.name, id: .sender.id,
  mid: .message_id, c: .content, reply: .reply_to, type: .msg_type}] | sort_by(.t)' \
  /tmp/paginate-out/<label>_page*.json > /tmp/paginate-out/<label>_flat.json

# Target 发言量 + 我跟他的 reply 链（冲突密度最高的区域）
jq --arg H "$TARGET_OU" --arg M "$MY_OU" '
  (map({(.mid): .}) | add) as $by_mid |
  [.[] | select(.reply != null) | . as $r | ($by_mid[.reply] // null) as $t |
    select($t != null) |
    select(($r.id == $M and $t.id == $H) or ($r.id == $H and $t.id == $M)) |
    {time: $r.t, from: $r.n, reply: $r.c, replied_to: $t.c, replied_from: $t.n}
  ] | sort_by(.time)
' /tmp/paginate-out/<label>_flat.json > /tmp/paginate-out/<label>_reply_chains.json
```

**别忘了代号/别名**：同事群里经常用数字代号 / 表情符号 / 昵称 / 英文名缩写互称（代号化后真名在群里出现频率大幅下降）。**光搜真名会漏一半信号**。Step 1 之前 / Step 4 分析时一定要**问用户这个人在相关群里有没有别称**，加进过滤词。

### 会议转录拉取（vc / minutes）

飞书会议的逐字稿由妙记（Minutes）产品托管，通过 `vc` 命名空间的三个命令索引到。人物画像里"线下尾巴"的核心数据源——**群里的人是他愿意让你看的形象，会议里才漏尾巴**。

**前置：scope 必须齐全。** 三个妙记 scope 缺一个 `vc +notes` 就会 403：`minutes:minutes:readonly` / `minutes:minutes.artifacts:read` / `minutes:minutes.transcript:export`。详见本文件"前置准备"的 auth 章节。

**三步链路：**

```bash
# === Step 1: 搜会议 ===
# 按参会人 + 时间范围找会议（人物路径的典型用法）
lark-cli vc +search \
  --participant-ids "<ou_target>" \
  --start "2026-03-01" \
  --end "2026-04-23" \
  --page-size 15 --format json \
  | jq '.data.meetings[] | {minute_token, title, start_time, duration}'
# 返回：title / minute_token / start_time / duration
# 也支持 --query <关键词>, --organizer-ids <ou>, --room-ids <room>
# 注意：--participant-ids 可传 "me" 代表当前用户

# === Step 2: meeting_id → minute_token (如果上一步已给 minute_token 可跳) ===
# 少数场景从 meeting_id 或 calendar_event_id 出发时才用
lark-cli vc +recording \
  --meeting-ids "<meeting_id>" \
  --format json
# 也支持 --calendar-event-ids

# === Step 3: 拉 notes (summary + todos + transcript) ===
lark-cli vc +notes \
  --minute-tokens "<token1>,<token2>" \
  --output-dir "tmp/transcripts/<minute_token>/" \
  --format json
# Transcript 落盘成 tmp/transcripts/<token>/transcript.txt（不是内联返回）
# JSON 返回索引：note_doc_token（AI 笔记 docx）/ verbatim_doc_token（逐字稿 docx）/ artifacts.transcript_file（本次下载的路径）
# --output-dir 强制指定输出目录——不给会污染 CWD（见下面"安全边界"段）
# 批量：逗号分隔 tokens，单次最多 N 个（首次实测时确认）
```

**Transcript 文件格式** (`transcript.txt`)：

```
<ISO 时间>|<总时长>

关键词:
<AI 抽取的关键词>

<说话人英文名>(<说话人中文名>) HH:MM:SS.ms 
<发言内容，原始逐字，含填充词/打断/重复>

<下一位说话人> HH:MM:SS.ms 
...
```

**关键坑：**
- **说话人是 display name，不是 open_id**。要严谨归属到 `persons/<X>.md` 得先 `contact +search-user --query "<中文名>"` 解析，避免同名误伤
- **`--output-dir` 必须显式给**。不给会落在 CWD 创建一个 `artifact-<title>-<token>/` 目录，污染仓库根。统一用 `tmp/transcripts/<minute_token>/`（`tmp/` 已 gitignore）
- **Skill 用完即删**。comm-brief 的 Step 9 要清理本次处理的 `tmp/transcripts/<token>/`，transcript 不做本地持久化
- **长会议 token 成本高**。1 小时的会议 transcript 约 10-30K tokens——让用户挑场次（AskUserQuestion），不要自动全量拉

**什么时候用（comm-brief 人物路径的数据源选择）：**
- 默认只拉消息即可
- 用户想看"完整画像" / 关心"线下尾巴" / 明确说"从最近那场会看 X 的表现" → 加会议转录这条路径
- 群路径（给群做画像）**不走转录**——转录是个人行为数据，不是群结构数据

### 其他已验证 shortcut（文档/日历等顺带记录）

```bash
lark-cli docs +create --title "<标题>" --markdown "<md 内容>"
lark-cli calendar +agenda
```

### 仍需 Phase 0 发现

**"@ 我的消息" / 今日 mentions** — 官方 `lark-im` skill 文档未明确封装这个场景。可选路径：
1. `lark-cli im +messages-search --help` 看是否有 `--mentioned-me` 之类 flag
2. 兜底方案：列所有 chat → 逐 chat 拉近期消息 → `jq` 本地过滤 mention 中含当前 user_id 的消息
3. 长期方案：`lark-event` skill 订阅实时事件（超出 Phase 0 scope）

### 原始 API 兜底（shortcut 不覆盖时）

飞书 Open API 常用消息端点参考：

| 端点 | 作用 |
|---|---|
| `GET /open-apis/im/v1/messages` | 列消息（需 `container_id_type` + `container_id`） |
| `GET /open-apis/im/v1/messages/<id>` | 单条详情 |
| `POST /open-apis/im/v1/messages` | 发消息 |
| `POST /open-apis/im/v1/messages/<id>/reply` | 回复 |
| `GET /open-apis/im/v1/chats` | 列我所在的群 |
| `POST /open-apis/contact/v3/users/batch_get_id` | 反查 user_id |

```bash
lark-cli api GET /open-apis/im/v1/messages \
  --params '{"container_id_type":"chat","container_id":"oc_xxx","page_size":20}' \
  --format json
```

---

## JSON 输出解析提示

- 飞书 Open API 外层统一包装：`{ "code": 0, "msg": "...", "data": { ... } }`，`code == 0` 才算成功
- 列表结果常在 `data.items[]`，分页游标在 `data.page_token`
- 消息文本在 `body.content`——通常是**另一层 JSON 字符串**，需要再 parse 一次：
  ```bash
  lark-cli api GET /open-apis/im/v1/messages --params '{...}' --format json \
    | jq '.data.items[] | {
        id: .message_id,
        sender: .sender.id,
        content_raw: .body.content,
        content: (.body.content | fromjson)
      }'
  ```
- `sender.id` 的 type 看 `sender.id_type`（`open_id` / `user_id` / `union_id`）——我们档案里存 `open_id`（`ou_` 前缀）

---

## 错误兜底

| 现象 | 处理 |
|---|---|
| `lark-cli: command not found` | `npm install -g @larksuite/cli` |
| `not authenticated` / `401` | `lark-cli auth login --recommend` |
| `403 / permission denied` | `lark-cli auth check --scope "<scope>"` 确认，缺的用 `lark-cli auth login --scope "<scope>"` 重新授权 |
| `429` 限流 | 等几秒重试；多次出现缩小 `page_size` |
| JSON 解析失败 | 把原始输出贴给用户看，**不要假装拿到了结构化数据** |
| Shortcut 不认 | 用 `lark-cli api` 直接走原始 API |

---

## 官方 AI Skills（分层定位）

Lark CLI 自带一套 Claude Code 可用的 skills，建议**安装作为通用数据接入层**：
```bash
npx skills add larksuite/cli -y -g
```

**分层关系：**
- **官方 skills（通用数据层）** — `lark-im` / `lark-contact` / `lark-event` 等，覆盖飞书通用 CRUD 场景（发消息、读消息、搜消息、列群、查联系人）
- **anti-olden（场景策略层）** — 在通用数据之上，专注"高情绪成本消息的分析 + 人物 Memory + 三种回复策略"

两者**组合关系**，不是竞品。本 cookbook 的 shortcut 命令正是官方 `lark-im` skill 所覆盖的——anti-olden 调 shortcut 等同于调用官方 skill 的能力。

**当官方 skill 和本 cookbook 冲突时：** 以 `lark-cli <cmd> --help` 或 `lark-cli schema` 的输出为准，更新 cookbook 对应条目。

**相关官方 skills：**
| Skill | 我们怎么用 |
|---|---|
| `lark-im` | 本 cookbook 的 `im +chat-messages-list` / `+messages-search` / `+messages-send` / `+messages-reply` 等 shortcut 全部来自这里 |
| `lark-contact` | 查联系人、反查 user_id；首次用 `lark-cli contact --help` 发现具体 shortcut |
| `lark-event` | 订阅实时事件（实时 mention 推送等，Phase 0 scope 外） |
| `lark-openapi-explorer` | 通用 API 发现——当上述都不覆盖时用它探 |

---

## 安全边界（再强调一次）

- **读**操作（GET API / list / search / schema 查看）→ 可以自由跑
- **写**操作（POST / PUT / DELETE / send / reply / create / update）→ **必须**在用户显式确认后才执行
- 确认前：跑 `--dry-run` 把完整 payload 贴给用户，等他说"发"再去掉 `--dry-run` 真跑
