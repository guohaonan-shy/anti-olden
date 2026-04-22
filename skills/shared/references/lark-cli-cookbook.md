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
lark-cli im +chat-search --keyword "<群名关键字>" --format json
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
