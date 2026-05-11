---
name: obsidian-workflow
description: 完整摄入流水线。输入原始资料路径，依次执行 vault 检查 → ingest → audit → field-update → overview-update。用法：/obsidian-workflow <原始资料路径> [--skip-audit] [--skip-field] [--skip-overview] [--ingest-only]
---

# Obsidian Workflow

完整知识摄入流水线，串联四个独立 skill。

```
vault-check → ingest → audit → field-update → overview-update
```

---

## CONFIG（使用前修改，或写入 CLAUDE.md）

```
VAULT       = /path/to/your/obsidian/vault
CONCEPTS    = $VAULT/Concepts
SOURCES     = $VAULT/Sources
FIELDS      = $VAULT/Fields
OVERVIEWS   = $VAULT/Overviews
TEMPLATES   = $VAULT/Templates
LOG         = $VAULT/log.md
INDEX       = $VAULT/index.md
```

---

> **CONFIG 说明**：各 skill 的 CONFIG 块是供 Claude 解析的键值表，不是 shell 脚本。`$VAULT/Concepts` 这类写法由 Claude 在执行时展开，实际运行 bash 命令时替换为完整路径并加引号。

## Step 0 — Vault 结构检查

检查 CONFIG 中所有路径是否存在：

```bash
for dir in $CONCEPTS $SOURCES $FIELDS $OVERVIEWS $TEMPLATES; do
  [ -d "$dir" ] || echo "MISSING: $dir"
done
```

- **全部存在** → 直接进入 Step 1
- **有缺失** → 列出缺失目录，询问用户：
  - 选 **自动创建** → `mkdir -p` 创建所有缺失目录，继续
  - 选 **手动处理** → 退出，提示用户先配置 CONFIG

---

## Step 1 — Ingest

调用 `obsidian-ingest` skill，输入：原始资料路径。

执行完毕后，解析 ingest 报告中的结构化字段：
- 读取 `DOC_NAME: <文档名>` → 作为 `$DOC_NAME` 传给后续步骤
- 读取 `DOC_PATH: <路径>` → 验证文件实际存在
- 新建概念卡列表（记录，最终报告用）

如果 ingest 输出多组 `DOC_NAME/DOC_PATH`（批量论文），对每组依次执行 Step 2-3，Step 4 最后统一执行一次。

---

## Step 2 — Audit

调用 `obsidian-audit` skill，输入：`$DOC_NAME`

执行完毕后：
- 若发现 ❌ Error → **暂停，列出错误，询问用户：**
  - 选 **手动修复**：修复后运行 `/obsidian-workflow <原路径> --skip-ingest` 从 audit 续跑
  - 选 **忽略继续**：记录 Error，继续执行后续步骤
- 若只有 ⚠️ Warning → 记录，自动继续
- 若全部 ✅ → 直接继续

---

## Step 3 — Field Update

调用 `obsidian-field-update` skill，输入：`$DOC_NAME`

执行完毕后，记录：
- 更新了哪个 Field（或新建了 Field）
- 补入了哪些链接

自动继续。

---

## Step 4 — Overview Update

调用 `obsidian-overview-update` skill，无需额外输入。

执行完毕后输出最终汇总报告。

---

## 最终报告

```
# Workflow Report: <原始资料名>
日期：YYYY-MM-DD

## Vault 检查
目录状态：全部就绪 / 新建 N 个目录

## 摄入
总结文档：[[文档名]]
概念卡：新建 N 个 / 补充 N 个

## 审查
OK: N / Warning: N / Error: N

## Field 更新
Field：[[Field名]]（更新 / 新建）

## Overview 更新
概念卡总数：N（A:N / B:N / C:N）
```

---

## 跳过规则

| 参数 | 效果 |
|------|------|
| `--skip-ingest` | 跳过 Step 1（需通过 `--doc-name=<文档名>` 显式指定要审查的总结文档） |
| `--skip-audit` | 跳过 Step 2 |
| `--skip-field` | 跳过 Step 3 |
| `--skip-overview` | 跳过 Step 4 |

示例：

```
# 修复 audit Error 后续跑
/obsidian-workflow --skip-ingest --doc-name=Paper_Smith2024_Keyword

# 只跑 Field + Overview 更新
/obsidian-workflow --skip-ingest --skip-audit --doc-name=Paper_Smith2024_Keyword

# 完整流程跳过 audit
/obsidian-workflow /path/to/material --skip-audit
```
