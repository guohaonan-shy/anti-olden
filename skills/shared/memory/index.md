# Memory 索引

## 设计原则

- **AI 和用户是共同作者。** AI 在每次交互后自动提取观察写入；用户随时手动修正或补充。两者同等权威，混在同一文件里，不区分来源——跟 Claude Code 的 memory.md 同一思路。
- **三层加载。** `user-profile.md` 始终加载 → `groups/{group}.md` 按当前群加载 → `persons/{person}.md` 按对话人加载。
- **人档案记稳定特征，群档案记场景特征。** "张三 ego 高、看数据"放 person file；"张三在跨部门群收着"放 group file。同一个人在不同群的"多面性"通过群档案索引。
- **渐进累积。** 第一次交互只写最小 frontmatter + 一条观察；不预填空 section。
- **可被推翻。** Memory 是"写入时的认知"，新消息明显冲突就更新，不死守旧结论。

---

## Memory 内容边界

Memory 文件（`user-profile.md` / `persons/*.md` / `groups/*.md`）的**正文**只写**自然语言描述**——我是谁、某人什么风格、这个群什么氛围。

**技术 ID（`lark_user_id` / `chat_id` / `app_id`）不进正文**，只作为结构化元数据进 **frontmatter**（见 templates/），或编码在文件名里。正文是供人读、供 Agent 理解语境的；ID 是路由用的机器字段。

**当前用户自己的身份不持久化**——运行时查 `lark-cli auth status --format json` 拿 `userOpenId`（详见 cookbook "查当前用户身份"）。不把 ou_id / app_id 写进 `user-profile.md`。

**正文 section 自由，frontmatter 字段固定。** Template 里列的 `## 沟通风格` / `## 群氛围` 等是**参考骨架，不是 schema**——每份档案可以按对方/该群的真实特点增删改 section（老板档案 vs 同事 vs 朋友 section 应不同）。AI 不要为"符合模板"强行填空或漏填；也不要在已有档案里乱改 section 结构——**沿用该文件既有的 section**，新观察无处可落就新开一个 section 跟用户确认。只有 frontmatter 机器字段（`lark_user_id` / `chat_id` / `last_updated`）是必守的。**拉取状态（`last_fetched_*`）不在 memory frontmatter 里**——归 `raw/chats/<chat_id>/_meta.json`，见 `docs/memory-architecture.md` §4.1。

---

## 运行时路由（不维护集中索引）

Skill 需要定位某条消息对应的 person/group 文件时：
- **按 sender 的 `lark_user_id` 查人物** → `glob persons/*.md` → 读每个文件 frontmatter 的 `lark_user_id` 字段匹配
- **按 chat_id 查群** → `glob groups/*.md` → 读 frontmatter 的 `chat_id` 字段匹配

不维护一个集中的路由表（容易跟实际文件漂移）。索引即文件 frontmatter。

---

## 所有 memory 写入的通用规则（**不要跳**）

无论是 comm-brief 的消息驱动沉淀、还是 comm-memory 的用户驱动 CRUD，**写入前必须**：

1. **整理拟写入 diff**（整段或整块增量）
2. **展示 diff 给用户**（格式化清晰，原样贴）
3. **调 `AskUserQuestion` 工具**（不是文本问）给 3 个选项：
   - **A. 写入** —— 按 diff 落
   - **B. 让我改一下** —— 用户口头说怎么改 → AI 按修改重新算 diff → 回到第 2 步再展示 → 再问 —— **循环直到 A 或 C**
   - **C. 不写** —— 跳过
4. **选 B 必须循环**：改几次都要重新展示 + 重新问。不能改完就静默写。

memory 是 AI + 用户共同作者的产物；用户必须有"我说了算 + 我能改"的纠偏权。

**为什么没有"只更 checkpoint"这个选项了？** 双层架构下 IO 状态（`last_fetched_*`）属于 raw/_meta.json，comm-brief 在 Step 3 拉消息时**就地**自动 bump，不需要用户确认。用户选 C（不写 memory）不影响下次增量起点——raw 已经吃过这批消息。

---

## 谁写什么（写权限边界）

**驱动源不同**：
- **comm-brief = 消息驱动**：从 `lark-cli` 拉到的消息里提炼观察 → 候选写入。主业之一就是沉淀到 memory
- **comm-memory = 用户驱动**：完全依赖用户口述。**不拉消息**，复合意图（如"基于最近聊天给 X 建档"）→ 建议用户先跑 `/comm-brief`

| 写入目标 | reply-coach | comm-brief（消息驱动） | comm-memory（用户驱动） |
|---|---|---|---|
| `user-profile.md`（正文） | ❌ | ❌（"我"不该从群聊反向推） | ✅ |
| `persons/*.md`（正文 + frontmatter） | ❌ | ✅（消息里观察到的人物特征 / 互动模式） | ✅ |
| `groups/*.md` → `群氛围` / `关键人物` / `注意事项` | ❌ | ✅（从消息观察） | ✅ |
| `groups/*.md` → `话题基线` / `近期简报` | ❌ | ✅（冷启动建基线、增量加条目） | ✅ |
| `raw/chats/<id>/_meta.json` → `last_fetched_*` | ❌ | ✅（拉消息时自动 bump，不走用户确认） | ❌（不拉消息） |

**所有写入都受上面"通用规则"约束**——经用户 ABCD 选 A 才写，选 C 要循环改。

为什么 comm-brief 也能写人档案？因为群聊本身就是观察人的主要场。"张三这两周在 X 群密集吐槽 Y" 这种观察直接写到 `persons/zhang-san.md` 的某个段更自然，而不是让用户事后用 comm-memory 口头再报告一遍。

---

## Checkpoint 归属于 raw/_meta.json

**不再在 memory frontmatter 里存 `last_processed_*`**。IO 状态（拉到哪里了）归 raw 层：

- `raw/chats/<chat_id>/_meta.json` 的 `last_fetched_at` / `last_fetched_message_id`
- comm-brief 拉消息时自动 bump，不走用户确认
- 冷启动触发条件不变：`_meta.json` 不存在 / `last_fetched_at` 距今 > 30 天 → 冷启动默认拉近 **14 天**

设计理由见 `docs/memory-architecture.md` §4.1。memory 文件专注承载"画像"，IO 状态不污染正文和 frontmatter。

---

## 命名约定

### 人物
- 默认：`persons/<name-pinyin>.md`（例：`zhang-san.md`）
- 同名冲突：`persons/<name-pinyin>_<ou_id 后 6 位>.md`（例：`zhang-san_abc123.md`）
- `lark_user_id` 是唯一主键，姓名仅用于显示

### 群组
- 默认：`groups/<group-name-pinyin>.md`（例：`chanpin-jishu-zhouhui.md`）
- 群名不明确时：`groups/<chat_id>.md`

---

## 检索顺序

处理一条消息时：
1. **始终加载** `user-profile.md`（不存在就按 comm-memory 的"首次建档"引导建）
2. 消息来自某个群 → glob `groups/*.md` → frontmatter 的 `chat_id` 匹配 → 命中就 Read；未命中时不自动建，等用户通过 comm-brief / comm-memory 触发
3. 按 sender 的 `lark_user_id` → glob `persons/*.md` → frontmatter 的 `lark_user_id` 匹配 → 命中就 Read；未命中同上
4. 建档基于 `templates/` 作为 schema 参考（只读），Write 到 `groups/` 或 `persons/` 下的新文件——**template 本身永不被覆盖**
