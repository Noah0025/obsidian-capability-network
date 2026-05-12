# Vault Config（obsidian-capability-network）

本文件定义 vault 路径和命名约定。把内容**放到你 AI 工具会自动读取的位置**，所有 skill 就能自动取到路径，无需每次调用重复指定。

| 你用的 AI 工具 | 放置位置（在 vault 根目录） |
|---|---|
| Claude Code | 复制为 `CLAUDE.md` |
| Cursor | 复制为 `.cursor/rules/vault-config.mdc` |
| Cline | 合并到 `.clinerules` 顶部 |
| Codex | 合并到 `AGENTS.md` 顶部 |
| 其他 | 调用 skill 时把这段 CONFIG 粘到 system prompt |

---

## 路径配置

```
VAULT      = /absolute/path/to/your-vault
CONCEPTS   = $VAULT/Concepts
SOURCES    = $VAULT/Sources           # 所有总结文档放这里，默认按 source_type 自动建子目录
FIELDS     = $VAULT/Fields
OVERVIEWS  = $VAULT/Overviews
TEMPLATES  = $VAULT/Templates
LOG        = $VAULT/log.md
INDEX      = $VAULT/index.md
```

> 默认行为：ingest 首次遇到新 `source_type` 时自动建 `Sources/<source_type>/` 子目录。
> 你也可以**提前自建子目录结构**，ingest 检测到现有目录会沿用而不重复创建：
> - 学生：`Sources/course/`、`Sources/paper/`、`Sources/project/`
> - 团队：`Sources/report/`、`Sources/meeting/`、`Sources/customer_call/`
> - 研究：`Sources/paper/`、`Sources/dataset/`、`Sources/experiment/`
>
> 子目录名 = `source_type` 字面量（小写下划线）。

> **说明**：这是 Claude 解析的键值表，不是 shell 脚本。`$VAULT/Concepts` 由 Claude 在执行时展开成完整路径。

---

## 命名约定（参考，可改）

| 层级 | 命名 | 示例 |
|------|------|------|
| Overview | `Overview_<主题>` | `Overview_Research` |
| Field | `Field_<领域>` | `Field_Statistics` |
| Concept/A | `A_<主题>` | `A_Neural_Networks` |
| Concept/B | `B_<机制>` | `B_Backpropagation` |
| Concept/C | `C_<公式/规则>` | `C_Chain_Rule` |
| Paper | `Paper_<FirstAuthor><YYYY>_<Keyword>` | `Paper_Smith2024_Keyword` |
| 其他 source | `<Type>_<Name>` 或自定义 | `Report_Q1_2026`、`Meeting_Kickoff` |

可以全用英文或全用本地语言，关键是 **vault 内保持一致**。

---

## Frontmatter 必填字段（按文档类型）

```yaml
# 概念卡
tags: [Concept/A]   # 或 Concept/B / Concept/C
aliases: [跨语言术语]

# Summary
tags: [Summary/<Type>]   # Type 用 TitleCase，如 Summary/Paper、Summary/Report、Summary/CustomerCall
source_type: paper       # 开放字段，用 lowercase 下划线：paper / course / report / meeting / customer_call / ...
date: YYYY-MM-DD
status: active

# Field
tags: [Field]

# Overview
tags: [Overview]
```

---

## 默认工作流

1. 喂资料：`/obsidian-workflow <资料路径>`
2. 定期健康检查：`/obsidian-audit --global`

更多细节看 `QUICKSTART.md` 和各 SKILL.md。
