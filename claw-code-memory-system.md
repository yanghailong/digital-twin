# claw-code 记忆系统架构分析

> claw-code 是 Claude Code 的 Rust 开源重写版本。本文档分析其"记忆系统"的实现机制。

## 整体架构

claw-code 的"记忆系统"**并不是一个独立的记忆存储引擎**，而是由两层机制构成：

1. **指令文件发现**（instruction files）— 已在 Rust 中实现的运行时层
2. **自主记忆**（auto memory）— 完全由系统提示词驱动的模型行为

原始 TypeScript 版本中的记忆子系统（`memdir/`、`SessionMemory/`）**尚未移植到 Rust**。

---

## 1. 指令文件发现（Instruction Files）

**代码位置**：`rust/crates/runtime/src/prompt.rs`

### 发现机制

从当前目录向上遍历祖先目录，依次查找以下文件：

- `CLAUDE.md`
- `CLAUDE.local.md`
- `.claw/CLAUDE.md`
- `.claw/instructions.md`

### 约束

| 约束项 | 限制值 |
|--------|--------|
| 单文件字符上限 | 4,000 字符 |
| 总字符上限 | 12,000 字符 |
| 去重方式 | 内容哈希去重，相同内容只保留一份 |

发现的指令文件内容作为 `# Claude instructions` 段落注入到系统提示词中。

### `/memory` 命令

仅作为只读查看器，列出已发现的指令文件（路径、行数、预览内容），不做任何写入操作。

---

## 2. 原始 TypeScript 记忆子系统（归档，未移植）

原始 Claude Code TypeScript 代码中有两个完整的记忆子系统，在 claw-code 中仅存为 `src/reference_data/` 下的 JSON 索引归档。

### `memdir/` 子系统 — 8 个模块

| 模块 | 功能 |
|------|------|
| `findRelevantMemories.ts` | 查找相关记忆 |
| `memdir.ts` | 核心记忆目录管理 |
| `memoryAge.ts` | 记忆时效管理 |
| `memoryScan.ts` | 记忆扫描 |
| `memoryTypes.ts` | 记忆类型定义（user / feedback / project / reference） |
| `paths.ts` | 路径解析 |
| `teamMemPaths.ts` | 团队记忆路径 |
| `teamMemPrompts.ts` | 团队记忆提示词 |

### `services/SessionMemory/` — 3 个模块

| 模块 | 功能 |
|------|------|
| `sessionMemory.ts` | 会话记忆 |
| `sessionMemoryUtils.ts` | 工具函数 |
| `prompts.ts` | 记忆提示词 |

### 其他

- `skills/bundled/remember.ts` — `/remember` 技能实现

---

## 3. "自主记忆"的工作原理

自主记忆**没有任何特殊的 API 或工具**，完全通过系统提示词驱动模型行为实现：

```
系统提示词嵌入 `# auto memory` 指令
    ↓
指令告诉模型何时应该保存记忆
（用户纠正行为、学到用户偏好等场景）
    ↓
模型使用 Write 工具将记忆写入 ~/.claude/projects/<path>/memory/
    ↓
MEMORY.md 作为索引文件
    ↓
下次会话启动时，MEMORY.md 内容被注入到上下文中
```

### 记忆类型

| 类型 | 用途 |
|------|------|
| `user` | 用户角色、目标、偏好、知识背景 |
| `feedback` | 用户对工作方式的纠正或确认 |
| `project` | 项目进展、目标、计划等非代码信息 |
| `reference` | 外部系统中的信息来源指针 |

### 记忆文件格式

每条记忆为一个独立的 markdown 文件，使用 frontmatter 元数据：

```markdown
---
name: 记忆名称
description: 一行描述，用于未来会话判断相关性
type: user | feedback | project | reference
---

记忆内容
```

---

## 4. 核心组件对照表

| 组件 | 实现方式 |
|------|----------|
| 存储 | 普通 markdown 文件 |
| 索引 | `MEMORY.md` 文件 |
| 读取触发 | 系统提示词指示模型加载 `MEMORY.md` |
| 写入触发 | 系统提示词指示模型在特定条件下调用 Write |
| 相关性判断 | 模型根据 `description` 字段自行判断 |
| 去重 | 模型被指示先检查是否已有同类记忆 |

---

## 5. 复用要点

要在其他项目中复用这套记忆系统，只需要三步：

1. **复制 `# auto memory` 相关的系统提示词片段** — 这是核心驱动力
2. **确保模型有文件读写能力** — Read / Write 工具
3. **在每次会话启动时把 `MEMORY.md` 的内容注入上下文** — 实现记忆持久化

### 关键结论

claw-code 的 Rust 运行时本身并没有增加额外的记忆逻辑。真正的"自主记忆"完全是 **提示词工程 + 文件系统** 的组合。模型被提示词"教会"了在合适的时机读写文件，从而实现了跨会话的记忆持久化。
