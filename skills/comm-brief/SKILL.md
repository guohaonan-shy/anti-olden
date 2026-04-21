---
name: comm-brief
description: 飞书沟通简报——汇总今天被@的消息、某个群最近动态、跟某人的近期互动模式。触发场景：用户说"今天被@了几次"/"这个群最近什么情况"/"跟某人最近怎么样"/"帮我梳理一下今天的消息"/"有哪些需要回的"。依赖 Lark CLI（`lark-cli`）。
---

# 沟通简报

## 你要做什么

帮用户快速了解飞书上的沟通全景或某个特定切面。**只做信息汇总和优先级标记**——不分析意图、不生成回复、不给策略。用户看完简报后如果要回某条，会去用 `reply-coach`。

## 工具权限

- **Bash** — 调 `lark-cli` 拉消息
- **Read** — 读 `../shared/memory/`（只读）
- **AskUserQuestion** — 确认关注范围、消歧

不需要 Write/Edit（不修改 memory）。不需要 WebSearch。

## 前置检查

1. `lark-cli --version`
2. `lark-cli auth status`

---

## 工作流（对话式）

### Step 1: 确认用户想看什么

用户可能说得很泛（"帮我看看今天的消息"）。先明确范围：

```
问："你想看哪个维度？"
  → 今日 @ 汇总（所有被 @ 的消息）
  → 某个群的动态（指定群）
  → 跟某人的近期互动（指定人）
  → 全部梳理一遍

如果用户指定了具体群或人，确认是哪个：
  "是'产品技术周会'那个群吗？"
```

### Step 2: 拉消息

Bash 调 `lark-cli`，命令查 `../shared/references/lark-cli-cookbook.md`。

按场景选不同路径：
- **今日 @**：参见 cookbook "仍需 Phase 0 发现"段落——搜 mention 或逐群拉
- **群动态**：`lark-cli im +chat-messages-list --chat-id <id> --format json`
- **人物互动**：`lark-cli im +messages-search` 按 sender 过滤

### Step 3: 加载 Memory（只读，增强判断）

- 始终读 `../shared/memory/user-profile.md`（知道用户角色，才能判断优先级）
- 按涉及的人读 `../shared/memory/persons/` 档案 → 标注"此人有档案记录"的附加信息
- 按涉及的群读 `../shared/memory/groups/` 档案 → 结合群氛围判断

**Memory 不存在时**不报错、不中断——没档案就没有附加标注，简报照出。

### Step 4: 输出简报

格式——简洁列表，按紧急度排序：

```
[需要回] 产品技术周会 — 张三
  "方案你再想想吧"——典型模糊否定，可能需要用就事论事回
  档案：此人 ego 高，上次用就事论事有效

[需要回] 李四（私聊）
  问了排期，等你确认
  无档案

[仅知晓] 全员群 — 王五
  通知下周五团建
  不需要回复

[可忽略] 产品技术周会 — 赵六
  在讨论别人的事，@你只是 FYI
```

**每条包含：**
- 紧急度标记：`需要回` / `仅知晓` / `可忽略`
- 群名或私聊 + 人名
- 消息摘要（一句话）
- （如有档案）关键特征提示

### Step 5: 收尾

```
简报输出后问："有哪条想展开看看？或者直接回哪条？"
  → 用户选了某条要回 → 提示"可以用 reply-coach 来帮你想怎么回"
  → 用户选了某人想了解更多 → 提示"可以用 comm-memory 看他的档案"
```

---

## 共享资源

- Lark CLI 命令：`../shared/references/lark-cli-cookbook.md`
- Memory（只读）：`../shared/memory/`
