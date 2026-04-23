# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目现状

设计权威来源是 `docs/anti-olden-project-doc.md`（v0.1，2026-04-16）。当用户请求偏离其中决策时，提示是否要同步更新文档。

**已完成：**
- `skills/anti-olden/` Phase 0 原型骨架（SKILL.md + references + prompts + memory 索引 + 人物档案模板）

**未完成：**
- `references/lark-cli-cookbook.md` 已对齐官方 README 和 `lark-im` skill 文档：二进制名（`lark-cli`）、认证流程、发现机制（`schema`）、输出格式、`--dry-run`、`--as user|bot`、shortcut 命令名（`+chat-messages-list` / `+messages-search` / `+messages-reply` / `+messages-send` / `+messages-mget` / `+chat-search`）。**但每个 shortcut 的具体 flag 还未实测**——Phase 0 首次使用时用 `lark-cli <cmd> --help` 确认并把完整形式覆盖回对应行
- "@ 我的消息" 场景官方 skill 未明确封装，Phase 0 需要发现路径（见 cookbook "仍需 Phase 0 发现"）
- 桌面端（Tauri v2）—— 用户自己设计交互和 UI，不要擅自开写
- `memory/persons/` 目录下无任何档案 —— 运行时按需累积

**当前阶段：内测自用。** 仓库保持私有，Skill 只作者自己装。分发/打包问题（公开 vs 私有、拆仓库、marketplace）延后到 Phase 0 验证完成再决定（见文档 §7.6）。写代码时**不要**预先加"防止 prompt 泄漏"的混淆、license 头、发行版隔离机制。

## 产品定位

Anti-Olden 是"职场沟通辅助 Agent"：当用户在飞书收到高情绪成本的消息时，Agent 负责**把情绪与策略分离**——不被激怒、只关注怎么措辞把事情往前推一步。项目内部代号"老登"指高沟通成本的协作对象（high ego / 习惯性否定 / 消耗对方情绪但不推进工作）。

**"参谋"而非"代笔"。** 调研显示重度 AI 辅助的消息真诚度只有 40-52%，低 AI 介入是 83%（详见 `docs/current-ai-usage-for-workplace-communication.md`）。产品定位是"帮你沟通得更好"而不是"替你沟通"——输出**建议框架**让用户修改后发，不是生成"直接粘贴的完美回复"。禁止生成公司腔客套话。

## 双轨架构（共享核心逻辑）

两条产品线**共用消息解析、人物画像、回复策略生成逻辑**，仅在 UI 层和 Memory 存储层分岔：

| 线 | 形态 | Memory | 优先级 |
|---|---|---|---|
| **Claude Code Skill** | 在 Claude 会话中通过 Skill 调用 | Skill 目录下的 markdown 文件 | 短期（Phase 0，1-2 周） |
| **桌面端 App** | Tauri v2，全局快捷键呼出 | SQLite + ChromaDB | 中期（Phase 1，4-6 周） |

## 核心架构约束

**Lark CLI 是唯一数据接入层。** 这是一条强约束，不要偏离：

- 不自建飞书 SDK、不包装 MCP。Skill 用 Claude Code 的 Bash 工具调 `lark-cli`；桌面端用 Rust `Command` 调同一个 CLI
- 认证由 `lark-cli auth login` 一次性处理（OAuth），Skill/App 不接触凭证
- 前置检查：`lark-cli --version`（是否安装）+ `lark-cli auth status`（是否登录）
- CLI 输出 JSON（`--format json`），程序侧解析

**分层定位：通用数据层 vs 场景策略层。**

- **官方 Lark skills**（`lark-im` / `lark-contact` / `lark-event` 等，通过 `npx skills add larksuite/cli -y -g` 安装）= 通用数据接入层，封装了飞书 CRUD 所有 shortcut（`im +chat-messages-list` / `+messages-search` / `+messages-reply` 等）
- **anti-olden** = 场景化策略层，做"高情绪成本消息的分析 + 人物 Memory + 三种回复策略"

**组合关系，不是竞品。** anti-olden cookbook 里的 shortcut 全部来自官方 `lark-im` skill——等于站在官方 skill 肩上，专注我们的策略层增值。

**写操作的安全边界。** Skill/App 可以读消息、读上下文，但**所有写操作**（发送消息、修改群设置等）必须经用户显式确认。发送前先 `--dry-run` 把 payload 给用户看一遍，确认后去掉 `--dry-run` 真跑。默认行为是"生成回复文本让用户自己复制粘贴"，不自动发送。

**一个 Skill，不拆分。** 不要把"飞书通用访问"拆成独立 Skill——官方已经有了（`lark-im` 等），anti-olden 只做策略 + 记忆这一层。

## 回复策略系统

三种预设策略，不要擅自增减；如需新策略先和用户对齐：

| 策略 | 适用 |
|---|---|
| **柔和推进** | 对方需要被尊重感、ego 很高 |
| **就事论事** | 对方习惯性否定但无实质论据 |
| **强势推回** | 对方在推卸责任或越界 |

对应 prompt 文件放 `shared/prompts/reply-soft.md` / `reply-factual.md` / `reply-push.md`。

## Skills 架构（三个 skill + 共享资源）

不是一个大 skill，而是三个独立 skill 各有明确触发场景，共享 memory / raw / references / prompts：

```
<repo root>/
└── skills/
    ├── reply-coach/SKILL.md       # 回复参谋：分析 + 策略 + 回复建议
    ├── comm-memory/SKILL.md       # 记忆管理：查看/编辑/新建 档案（纯口述，不拉消息）
    ├── comm-brief/SKILL.md        # 沟通简报：拉消息/转录 → 写 raw → 沉淀 memory
    └── shared/
        ├��─ references/
        │   ├── lark-cli-cookbook.md    # Lark CLI 命令配方
        │   └── olden-patterns.md       # 行为模式速查
        ├── prompts/
        │   ├── analyze.md              # 消息分析结构
        │   ├── analyze-transcript.md   # 会议转录分析结构
        │   ├── reply-soft.md           # 柔和推进
        │   ├── reply-factual.md        # 就事论事
        │   └── reply-push.md           # 强势推回
        ├── memory/                     # 提炼观察（入仓：templates + index）
        │   ├── templates/              # 建档模板
        │   ├── index.md                # 索引 + 设计原则
        │   ├── user-profile.md         # 用户画像（gitignore）
        │   ├── persons/{person}.md     # 人物档案（gitignore）
        │   └── groups/{group}.md       # 群组上下文（gitignore）
        └── raw/                        # 原始数据缓存（全量 gitignore，跨会话持久）
            ├── chats/<chat_id>/        # _meta.json + <YYYY-MM>.ndjson + threads/
            ├── attachments/<file_key>  # 图片/文件/音频（默认下载）
            └── transcripts/<token>/    # 会议逐字稿 + _meta.json
```

**关键分工（双层 memory 架构，详见 [docs/memory-architecture.md](docs/memory-architecture.md)）：**
- **reply-coach** 读 memory + raw 拿画像和原始上下文，不写——回复辅助跟记忆管理解耦
- **comm-memory** 读写 memory，**不拉消息也不碰 raw**——纯口述 CRUD
- **comm-brief** 读写 memory + 拉 lark-cli → 追加 raw → 基于 raw 提炼观察 → 沉淀 memory

## Memory 架构（双层：Raw + Memory）

**权威设计来源：`docs/memory-architecture.md`**（v0.1，2026-04-23）。下面是精简总览，细节以设计文档为准。

### 两层分工

| 层 | 内容 | 生命周期 | 路径 |
|---|---|---|---|
| **Memory** | 提炼后的观察（自然语言），供 LLM 读、供用户审 | 长期，AI + 用户共同维护 | `skills/shared/memory/` |
| **Raw** | 原始消息 / 附件 / 转录（按 chat 按月分桶的 ndjson + 文件） | 长期持久，跨会话复用 | `skills/shared/raw/`（gitignore） |

**Memory 设计原则（不变）：**
- AI + 用户共同维护，两者同等权威
- 三层加载：`user-profile.md` 始终 → `groups/{group}.md` 按群 → `persons/{person}.md` 按人
- 人档案记稳定特征 + 互动模式，群档案记场景特征；同一人"多面性"通过群档案索引
- 渐进累积，可被推翻

**Memory 三层结构：**

| 层 | 文件 | 内容 |
|---|---|---|
| 用户画像 | `shared/memory/user-profile.md` | 我的角色 / 沟通风格 / 禁忌 / 默认策略偏好 |
| 群组上下文 | `shared/memory/groups/{group}.md` | 群氛围 / 关键人物在此群的表现 / 注意事项 |
| 人物档案 | `shared/memory/persons/{person}.md` | 沟通风格 / 雷区 / **互动模式** / 有效策略 / 关系备注 |

**重名处理：** `persons/<pinyin>_<ou_id 后 6 位>.md`。`lark_user_id` 唯一主键。详见 `shared/memory/index.md`。

### Raw 层要点

- **Checkpoint 归属**：`last_fetched_at` / `last_fetched_message_id` 只放 `raw/chats/<id>/_meta.json`，**不在 memory frontmatter 里**。IO 状态归 raw，画像归 memory
- **按月分桶**：`raw/chats/<chat_id>/<YYYY-MM>.ndjson`；主消息去重靠 `message_id`
- **Thread 独立存**：`raw/chats/<chat_id>/threads/<thread_id>.ndjson`，lazy + 按需缓存
- **附件默认下载**：`raw/attachments/<file_key>.<ext>`，skill 读图用 `Read` 走视觉
- **Transcript 跨会话持久**：`raw/transcripts/<minute_token>/`，不删
- **GC**：Phase 0 不做。文本 + 附件长期 1-2 GB 内，超过再按 chat_id 按月做归档策略
- **追加协议的 CLI 实现**在 `shared/references/lark-cli-cookbook.md` "Raw 数据缓存落盘协议" 章节

## 开发路线（当前阶段）

现在是 **Phase 0: Skill 原型**。关键验证点：
1. Claude 能否正确理解自然语言并翻译成 Lark CLI 命令
2. CLI 返回的 JSON 是否足够让 Claude 理解消息上下文
3. 生成的回复建议是否真的"比用户自己想得好"

Phase 1（桌面端 Tauri v2）和 Phase 2（ChromaDB 语义检索 + 自动 Memory 提取）都还未开始——不要提前写这两阶段的代码，除非用户明确要求切换阶段。

## 技术选型要点

- **桌面端框架：Tauri v2**（不是 v1，不是 Electron）。理由：体积小、Rust 后端、ACL 安全模型
- **前端框架：React / Vue / Svelte 三选一**，尚未定。如需选，Svelte 是文档里偏好的方向（工具型产品 UI 简单）
- **LLM：provider-agnostic 抽象层**。定义统一 `LLMService` 接口，v1 先用 Claude API 跑通（与 Skill 路线一致），后续可扩展 OpenAI / 本地模型
- **桌面端 Memory：SQLite + ChromaDB 自建**（都有 Rust binding），不用 Mem0（Python 生态，跨语言集成成本高）

## 命名约定

- "老登" 是**项目内部代号**，出现在内部文档和代码注释里；**不要**暴露到最终用户可见的 UI 文案
- 用户角度的术语：沟通参谋、回复建议、策略、人物档案、工作会话
