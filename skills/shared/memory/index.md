# Memory 索引

## 设计原则

- **AI 和用户是共同作者。** AI 在每次交互后自动提取观察写入；用户随时手动修正或补充。两者同等权威，混在同一文件里，不区分来源——跟 Claude Code 的 memory.md 同一思路。
- **三层加载。** `user-profile.md` 始终加载 → `groups/{group}.md` 按当前群加载 → `persons/{person}.md` 按对话人加载。
- **人档案记稳定特征，群档案记场景特征。** "张三 ego 高、看数据"放 person file；"张三在跨部门群收着"放 group file。同一个人在不同群的"多面性"通过群档案索引。
- **渐进累积。** 第一次交互只写最小 frontmatter + 一条观察；不预填空 section。
- **可被推翻。** Memory 是"写入时的认知"，新消息明显冲突就更新，不死守旧结论。

---

## 用户画像

| 文件 | 说明 |
|------|------|
| `user-profile.md` | 我是谁、我怎么说话、默认策略偏好（**始终加载**） |

---

## 人物档案

| 姓名 | 文件 | lark_user_id | 最近更新 | 备注 |
|------|------|--------------|----------|------|
| _(使用中逐步累积)_ | | | | |

---

## 群组上下文

| 群名 | 文件 | chat_id | 最近更新 | 备注 |
|------|------|---------|----------|------|
| _(使用中逐步累积)_ | | | | |

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
