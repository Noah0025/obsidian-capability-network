---
name: obsidian-overview-update
description: 读取所有 Field 文档，重新计算量化指标，更新全部 Overview 文档（全局知识地图）。用法：/obsidian-overview-update
---

# Obsidian Overview Update

读取全部 Field，重新统计量化数据，更新 `$OVERVIEWS/` 下的所有 Overview 文档。

---

## CONFIG（使用前修改，或写入 CLAUDE.md）

```
VAULT      = /path/to/your/obsidian/vault
CONCEPTS   = $VAULT/Concepts
SOURCES    = $VAULT/Sources
FIELDS     = $VAULT/Fields
OVERVIEWS  = $VAULT/Overviews
TEMPLATES  = $VAULT/Templates
```

---

## Step 0 — 收集全局数据

1. 读取所有 `$FIELDS/` 下的 Field 文档，提取每个 Field 的：
   - 引用的来源列表（从 **真实链接** 提取）
   - 关联的概念卡数量（统计所有引用来源在 `$CONCEPTS` 中的卡片引用数）
2. 全局统计：
   ```bash
   # 概念卡总数
   find "$CONCEPTS" -name "*.md" | wc -l
   # A/B/C 分布
   grep -rl "Concept/A" "$CONCEPTS" | wc -l
   grep -rl "Concept/B" "$CONCEPTS" | wc -l
   grep -rl "Concept/C" "$CONCEPTS" | wc -l
   # 总结文档数（按 source_type 通用扫描）
   grep -rhoE "Summary/[A-Za-z_]+" "$VAULT" | sort | uniq -c
   ```

---

## Step 1 — 更新每个 Overview

遍历 `$OVERVIEWS/` 下的所有 Overview 文档，逐个读取并更新。

### 1a. Field 引用完整性
- 检查所有 Field 是否在 Overview 的 **真实链接** 中被引用
- 新 Field 未引用 → 补入 **真实链接** + 在知识框架叙述中补一句说明

### 1b. 量化数据表
量化表用自动生成标记包裹，更新时只替换此区块，不影响区块外的用户内容：

```markdown
<!-- AUTO-GENERATED:START — 由 obsidian-overview-update 维护，请勿手动编辑此区块 -->
| Field | 概念卡数 | 来源数 |
|-------|---------|--------|
| Field_Name | N | N |
| ...   | ...     | ...    |
| **总计** | **N** | **N** |
<!-- AUTO-GENERATED:END -->
```

结构特征更新（同样用 AUTO-GENERATED 标记包裹）：
- 概念卡 A:B:C 分布
- 总结文档类型分布

### 1c. 能力地图
- 从各 Field 的"能力地图"章节汇总
- 检查 Overview 的能力矩阵是否遗漏新能力项
- 有遗漏 → 补入对应分类

### 1d. 知识缺口（如有此章节）
- 对比 Field 变更前后：新增的 Field / 来源是否填补了已列的缺口
- 已填补 → 标记或删除
- 新发现的缺口（从新 Field 的跨领域关联推断）→ 补入

---

## Step 2 — 验证

1. **Wikilink 完整性**：提取所有 Overview 的 wikilink，`find` 验证目标存在
2. **双向链接**：每个 Field 都被至少一个 Overview 引用

---

## Step 3 — 输出报告

```
# Overview Update Report
日期：YYYY-MM-DD

## 全局统计
概念卡：N（A:N / B:N / C:N）
总结文档：N
Field：N

## 更新内容
### Overview_<Name>
- 量化表：已更新
- 新 Field 引用：+N
- 能力地图：+N 项

## Wikilink 验证
断裂：0 / N 条待处理
```
