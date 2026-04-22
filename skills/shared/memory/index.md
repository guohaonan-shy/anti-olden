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

---

## 运行时路由（不维护集中索引）

Skill 需要定位某条消息对应的 person/group 文件时：
- **按 sender 的 `lark_user_id` 查人物** → `glob persons/*.md` → 读每个文件 frontmatter 的 `lark_user_id` 字段匹配
- **按 chat_id 查群** → `glob groups/*.md` → 读 frontmatter 的 `chat_id` 字段匹配

不维护一个集中的路由表（容易跟实际文件漂移）。索引即文件 frontmatter。

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
1. **始终加载** `user-profile.md`
2. 如消息来自某个群 → 按 `chat_id` 在群组表查 → 加载 `groups/{file}.md`；没有则用 `templates/group-profile.md` 新建
3. 按对方 `lark_user_id` 在人物表查 → 加载 `persons/{file}.md`；没有则用 `templates/person-profile.md` 新建
4. 新建的文件 → 回来更新本索引对应表
