# anti-olden Skills

职场沟通辅助 Agent 的 Claude Code skills——从飞书消息里观察 → 辅助沉淀 memory → 帮用户想怎么回。

**产品定位**：参谋而非代笔。禁止生成"直接粘贴的完美回复"，只给用户**建议框架**让他改后再发。详见项目根目录 `CLAUDE.md` + `docs/anti-olden-project-doc.md`。

## 三个 skill 的分工

| Skill | 流程体量 | 典型场景 | 数据源 | 写 memory |
|---|---|---|---|---|
| **comm-brief** | **重流程** | "给 X 建档"/"xx 群最近啥情况"（只给对象，没给描述） | 消息（lark-cli 拉）+ 用户口述 | ✅ 用户审+改+落 |
| **comm-memory** | **轻流程** | "记一下 X 配合得好"/"看看 X 档案"/"改我的风格"（口述已自足） | 纯用户口述，**不拉消息** | ✅ 用户审+改+落 |
| **reply-coach** | 独立 | "帮我回这条" | 消息 + 读 memory | ❌ 只读 |

**分界线**（关键）：

| 用户说什么 | 路由到 |
|---|---|
| "给 X 建档"（只给对象） | comm-brief（会去拉消息分析） |
| "X 爱引用数据，不喜欢 emoji，给他建档"（口述自足） | comm-memory（直接接口述） |
| "xx 群最近啥情况" | comm-brief |
| "记一下 X 今天配合得好" | comm-memory |
| "看看 X 档案" | comm-memory（纯读） |
| "帮我回这条" | reply-coach |

**comm-brief 的关键是"消息 + 口述融合"**——AI 拉消息出 draft 后会**主动问**你"还有啥认知要加的"（比如汇报线、私下八卦、你对他的既往判断），这些 AI 从消息看不到。

## 目录结构

```
skills/
├── reply-coach/SKILL.md       # 回复参谋
├── comm-brief/SKILL.md        # 消息驱动的 memory 沉淀
├── comm-memory/SKILL.md       # 用户驱动的 memory CRUD
└── shared/
    ├── references/
    │   ├── lark-cli-cookbook.md    # Lark CLI 命令配方
    │   └── olden-patterns.md       # 行为模式速查
    ├── prompts/
    │   ├── analyze.md              # 消息分析结构（reply-coach 用）
    │   ├── reply-soft.md           # 柔和推进
    │   ├── reply-factual.md        # 就事论事
    │   └── reply-push.md           # 强势推回
    └── memory/
        ├── index.md                # Memory 设计文档 + 通用写入规则
        ├── templates/              # 只读 schema，所有 skill 建档时参考，永不覆盖
        │   ├── user-profile.md
        │   ├── person-profile.md
        │   └── group-profile.md
        │
        │ ── 以下为运行时本地状态，.gitignore 外不入仓库 ──
        ├── user-profile.md         # 我自己的画像
        ├── persons/<pinyin>.md     # 每人一个文件
        └── groups/<pinyin>.md      # 每群一个文件
```

**`shared/memory/` 下的运行时文件（`user-profile.md` / `persons/` / `groups/`）都在 `.gitignore` 里**——每台机器 / 每个用户本地累积，不共享。仓库里只有 `index.md`（设计文档）和 `templates/`（schema）。

## 前置依赖

```bash
# 安装 Lark CLI（唯一数据接入层）
npm install -g @larksuite/cli

# 首次配置——创建自建应用（需要在飞书开放平台后台审批通过后才能用 OAuth）
lark-cli config init --new

# OAuth 登录（--recommend 一次拿常用 scope）
lark-cli auth login --recommend

# 验证
lark-cli auth status
```

三个 skill 本身不碰凭证——`lark-cli` 自己管本地 token。

## Memory 系统的核心约束

**两条全局规则**（详见 `shared/memory/index.md`）：

1. **正文只写自然语言描述**。技术 ID（`lark_user_id` / `chat_id`）放 frontmatter 或文件名。当前用户自己的身份不持久化——运行时查 `lark-cli auth status`。
2. **所有 memory 写入前都走"通用确认循环"**——展示 diff → 调 `AskUserQuestion` 工具给 ABCD 4 选项（含 C=编辑）→ 选 C 要循环改直到用户选 A 或 D。没有静默写入。

## 其他约束（产品边界）

- **参谋不代笔**：回复只给 3-5 行建议框架，用 `[这里换你自己的说法]` 留空位让用户填。禁止公司腔（"非常感谢" / "烦请" / "如有任何问题"）。
- **写操作必须经用户显式确认**。飞书消息发送 → 先 `--dry-run` 给用户看 payload → 用户说"发"去掉 `--dry-run` 真跑。默认是"让用户自己复制粘贴"。
- **"老登"是内部代号**——只在内部文档 / 代码里出现，**不暴露到用户最终看到的文案里**。

## 测试 / 迭代

Phase 0 内测自用。仓库私有，skill 只作者自己装。通过 `.claude/skills` 软链让 Claude Code 发现本目录的 skill。

分发问题（共享 app / marketplace / 打包）延后到 Phase 0 验证完再定——见 `docs/anti-olden-project-doc.md` §7.6。
