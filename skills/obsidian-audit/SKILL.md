---
name: obsidian-audit
description: 验证 Obsidian vault 的完整性和正确性。两种模式：单文档模式验证一个总结文档及其关联概念卡；全局模式扫描全部概念卡的结构合规性。用法：/obsidian-audit <文档名> 或 /obsidian-audit --global
---

# Obsidian Audit

验证 ingest 是否完整执行、文档结构是否合规。发现问题直接修复，复杂问题输出建议。

---

## CONFIG（使用前修改，或写入 CLAUDE.md）

```
VAULT      = /path/to/your/obsidian/vault
CONCEPTS   = $VAULT/Concepts
FIELDS     = $VAULT/Fields
OVERVIEWS  = $VAULT/Overviews
TEMPLATES  = $VAULT/Templates
SOURCES    = $VAULT/Sources           # 总结文档根目录（含任意子目录）
```

---

## 模式判断

- `$1` 为 `--global` → **全局模式**（Step G1–G5）
- `$1` 为文档名 → **单文档模式**（Step S1–S5）

---

# 单文档模式（/obsidian-audit <文档名>）

## Step S1 — 定位与读取

1. 在 `$SOURCES/` 下递归搜索文件名匹配的 `.md` 文件。
2. 读取全文，从 frontmatter 的 `source_type` 字段确定文档类型（开放字段，如 `paper`/`course`/`report`/`meeting`/...）。
3. 读取对应模板（`$TEMPLATES/summary.md`）。

---

## Step S2 — 模板合规性（✅ / ⚠️ / ❌）

### Frontmatter
- `tags:` 含正确 `Summary/<Type>` → ✅ 否则 ❌
- `status:` 字段存在且非空 → ✅ 否则 ⚠️
- `date:` 字段存在 → ✅ 否则 ⚠️

### 文档语言
总结文档主体语言应与原始资料一致。

### 章节结构

| 必需章节 | 检查方式 |
|---------|---------|
| 知识框架 / 核心内容 | 含 ≥1 个 `[[wikilink]]` |
| 核心能力 / 收获 | 含 `- [ ]` checklist 或对等结构 |
| 相关 | 含 **真实链接** 子节 |
| 参考来源 | 存在且非空 |

---

## Step S3 — 关联概念卡完整性

对总结文档引用的每张概念卡：
1. `find` 验证文件存在 → ❌ 断裂链接
2. `tags` 含 `Concept/A`、`Concept/B` 或 `Concept/C` → ❌ 缺失
3. `## 相关` 含 **真实链接** 子节 → ⚠️ 缺失
4. 卡片内 wikilink 可解析（目标文件存在）→ ❌ 断裂

---

## Step S4 — Field 覆盖

检查该总结文档是否被至少一个 `$FIELDS/` 下的文档引用。
- 未被引用 → ⚠️ 建议运行 `/obsidian-field-update`

---

## Step S5 — 报告 + 修复

**直接修复**：frontmatter 补正、`## 相关` 结构、断裂 wikilink（目标明确时）。

**输出建议**：缺失概念卡候选、Field 覆盖缺口。

```
# Audit: <文档名> (<类型>)
日期：YYYY-MM-DD

## 模板合规性
## 概念卡完整性
## Field 覆盖
## 统计（OK: N / Warning: N / Error: N）
## 建议操作
```

---

# 全局模式（/obsidian-audit --global）

Bash 优先扫描全部概念卡，只读取被标记的卡片。

## Step G1 — Tags 检查

```bash
grep -rL "Concept/" "$CONCEPTS" --include="*.md"
```
修复：根据路径深度推断级别（A/B/C），补入 frontmatter。

## Step G2 — 重复文件名

```bash
find "$CONCEPTS" -name "*.md" -exec basename {} \; | sort | uniq -d
```
输出重复列表，建议合并方向（不自动合并）。

## Step G3 — Wikilink 完整性

```bash
grep -rho '\[\[[^]]*\]\]' "$CONCEPTS" --include="*.md" | \
  perl -ne 'while (/\[\[([^\]|#]+)/g) { print "$1\n" }' | sort -u
```
逐一 `find` 验证目标存在。近似名直接修复，无法确定的输出到报告。

## Step G4 — ## 相关 结构

```bash
grep -rL "\*\*真实链接\*\*" "$CONCEPTS" --include="*.md"
```
修复：补全缺失的 **真实链接** 子节。

## Step G5 — 孤立卡（零入链）

```bash
# 差集：所有卡片名 vs 全 vault wikilink 目标
```
输出孤立卡列表，建议在上游父卡或总结文档中补入链接。

## Step G6 — 报告

```
# Global Audit Report
日期：YYYY-MM-DD
概念卡总数：N

| 检查项 | 发现 | 已修复 | 待处理 |
|--------|------|--------|--------|
| Tags 缺失 | N | N | N |
| 重复文件名 | N | — | N |
| 断裂 wikilink | N | N | N |
| 相关结构缺失 | N | N | N |
| 孤立卡 | N | — | N |
```
