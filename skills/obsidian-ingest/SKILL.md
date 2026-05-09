# Obsidian Ingest（材料摄入）

输入一个路径，自动判断资料类型，生成对应的概念卡和总结文档。完成后输出摄入报告。

---

## CONFIG（使用前修改）

```
VAULT      = /path/to/your/obsidian/vault
CONCEPTS   = $VAULT/Concepts          # 概念卡根目录
COURSES    = $VAULT/Courses           # 课程总结目录
LABS       = $VAULT/Labs              # 实验记录目录
PROJECTS   = $VAULT/Projects          # 课程项目目录
THESIS     = $VAULT/Thesis            # 论文目录
INTERNSHIP = $VAULT/Internship        # 实习目录
ARTICLES   = $VAULT/Articles          # 论文/文章目录
TEMPLATES  = $VAULT/Templates         # 模板目录
```

> Tip: 可将此 CONFIG 块复制到 `CLAUDE.md`，让 Claude 自动读取，无需每次手动指定。

---

## Step 0 — 类型判断与目标确认

1. 输入 `<path>` 为路径。若不存在，提示并退出。
2. `ls` 盘点内容。
3. **自动判断类型**（按优先级匹配）：

| 判断条件 | 类型 | 输出目录 | 命名规则 |
|---------|------|---------|---------|
| 路径是单个 `.pdf` 文件 | **Paper** | `$ARTICLES` | `Paper_AuthorYear_Keyword` |
| 文件夹内只有论文 PDF（无课件/PPT/多章节结构） | **Paper** | `$ARTICLES` | 同上，逐篇处理 |
| 路径含 `Masterarbeit` / `毕业设计` / `本科设计` / `thesis` | **Thesis** | `$THESIS` | `Thesis_Degree_School` |
| 路径含 `实习` / `internship` | **Internship** | `$INTERNSHIP` | `Internship_Org_Year` |
| 路径含 `课程设计` / `Projekt` / `project` | **Project** | `$PROJECTS` | `Proj_ProjectName` |
| 路径含 `实验` / `lab` | **Lab** | `$LABS` | `Lab_LabName` |
| 路径在课程资料目录下 | **Course** | `$COURSES` | `Course_CourseName` |
| 以上均不匹配 | 提示用户确认类型 | — | — |

4. 检查对应总结文档是否已存在 → 已存在则切换**补充模式**（只补概念卡和缺失章节）。

---

## Step 1 — 提取原始资料内容

### PDF 处理策略

**先尝试文字提取**：
```bash
pdftotext "file.pdf" - 2>/dev/null | head -100
```
- 有实质内容 → 直接用
- 空或乱码 → 扫描件，进入 OCR
- "encrypted" / "permission" → 标记加密，仅用文件名

**扫描件 OCR**（每次 10-15 页，处理完立即删临时文件）：
```bash
pdftoppm -r 150 -f <start> -l <end> "file.pdf" /tmp/ocr_tmp
for f in /tmp/ocr_tmp-*.ppm; do ocr_tool "$f" 2>/dev/null; echo "---"; done
rm -f /tmp/ocr_tmp-*.ppm
```

优先处理目录页（前 5-15 页）获取章节结构。

### OCR 结果处理

OCR 原始输出常有错字、断行——根据上下文智能推断：
- `"扩散模 型"` → `扩散模型`
- `"Re ynolds"` → `Reynolds`
- 无法推断的段落直接跳过

### 非 PDF 文件

- PPT/PPTX：用文件名推断话题
- DOCX：尝试 `textutil -convert txt`（macOS）或 `pandoc` 提取
- 其他：仅用文件名

---

## Step 2 — 概念提取与匹配

1. 从章节结构和内容中提取候选概念列表（有公式/定量内容的 B/C 级概念）
2. 对每个候选概念，用 `find $CONCEPTS -name "<概念名>.md"` 匹配已有卡片
3. 三分支处理：

| 匹配结果 | 动作 |
|---------|------|
| **已有且内容充分** | 跳过，记录到报告 |
| **已有但缺少本课视角** | 读取卡片，在 `## 各来源应用` 中补入本课引用 |
| **不存在且有公式/定量内容** | 新建 B 或 C 卡（见 Step 3） |

**跳过条件（不建卡）**：
- 纯叙述性背景（无公式/参数表）
- 不到一段的常识
- 软件操作步骤
- 与父 B 卡无本质区别的内容

---

## Step 3 — 创建概念卡

参照 `templates/concept_b.md` 或 `templates/concept_c.md`：

```markdown
---
tags:
  - Concept/B  # 或 Concept/C
aliases:
  - English Term
---

# 概念名

一句话定义。

---

## 原理

（机制/公式/参数表）

---

## 相关

- [[父概念]] — 所属机制
- [[来源课程/文章]] — 引入来源
```

**放置规则**：
- 检查是否有合适的父文件夹（同领域的 A 或 B 卡文件夹）
- 有 → 放入该文件夹
- 无 → 创建 `$CONCEPTS/概念名/概念名.md`

**Aliases 规则**：
- 有标准跨语言术语 → 补充（有几个补几个）
- 纯计算步骤类 → 不补
- 不确定 → 不猜

---

## Step 4 — 生成总结文档

按对应类型的模板生成总结文档，包含：
- frontmatter（tags、date、status 等）
- 知识框架（用 `[[wikilink]]` 引用本来源涉及的概念卡）
- 核心能力 checklist（5-8 条自测项）
- 来源衔接（前置/后续链接）
- 相关链接
- 参考来源

**文档语言**：与原始资料一致。

---

## Step 5 — 输出报告

```
# Ingest Report: <来源名>
路径：<path>
日期：YYYY-MM-DD

## 文件盘点
| 文件类型 | 数量 | 处理方式 |
...

## 概念卡
新建：N 个（附列表）
补充：N 个（附列表）
跳过：N 个

## 总结文档
文件：<路径>
状态：新建 / 补充

## 后续建议
- 检查概念卡链接完整性
- 更新 Field 和 Overview 文档
```

---

# Paper 类型专用步骤

当 Step 0 判断类型为 Paper 时，跳过课程模式的 Step 1-4，改用以下步骤。

---

## Step P0 — 文本提取

1. 单个 PDF → 处理 1 篇；文件夹 → `ls *.pdf`，逐篇处理。
2. 对每个 PDF：`pdftotext` 提取全文（论文通常是文字型 PDF，不需要 OCR）。
3. 若提取失败（扫描件）→ OCR 流程（同课程模式 Step 1）。

---

## Step P1 — 论文信息提取

从 PDF 全文中提取：
- **标题**（Title）
- **作者列表**（标注自己的位置：第一/通讯/共同作者）
- **期刊/会议名 + 年份**
- **DOI**
- **Abstract**
- **关键词**（Keywords）

确定命名：`Paper_FirstAuthorYear_Keyword`（如 `Paper_Smith2024_UrbanFlood`）。

检查 `$ARTICLES/` 中是否已存在同名文件 → 已存在则切换补充模式。

---

## Step P2 — 内容深度提取

逐章节读取（Introduction → Methods → Results → Discussion → Conclusion）：

1. **研究问题**：从 Introduction 最后一段提取
2. **方法**：提取数据来源、模型/框架名称、分析工具
3. **核心发现**：从 Results/Conclusion 提取 3-5 条带数值的关键结论
4. **局限性**：从 Discussion/Limitations 提取

对方法和发现中的关键概念，执行概念匹配（同课程模式 Step 2）：
- 已有 → 在 Paper 总结中直接 `[[wikilink]]` 引用
- 不存在且有独立方法论价值（论文提出的新模型/指标）→ 新建概念卡
- 通用方法 → 只引用已有卡，不新建

**论文模式的建卡门槛比课程模式更高**：只有论文提出的**原创方法/模型/指标**才建新卡，已有的通用概念只链接。

---

## Step P3 — 生成 Paper 总结卡

参照 `templates/summary.md`（source_type: paper），放入 `$ARTICLES/`：

```markdown
---
tags:
  - Summary/Paper
citation_key: AuthorYear
doi: <DOI>
date: YYYY
status: active
---

# 论文标题

作者列表 · 期刊/会议 · 年份

---

## TL;DR

3 句话：① 研究问题 ② 核心方法 ③ 最重要的结论（带数值）

---

## 研究问题

...

## 方法

数据来源、研究设计、分析工具。用 [[wikilink]] 引用概念卡。

## 核心发现

3-5 条带数值的关键结论。

## 对我的价值

- [ ] 方法可借鉴 — ...
- [ ] 数据可复用 — ...

---

## 相关

- [[相关概念卡]] — 方法/理论
- [[相关 Field]] — 所属领域

## 参考来源

- DOI: <DOI>
```

---

## Step P4 — 输出报告

```
# Paper Ingest Report
日期：YYYY-MM-DD

## 论文列表
| # | 命名 | 标题 | 期刊 | 年份 | 概念卡 |
...

## 概念卡
新建：N 个（附列表）
引用已有：N 个
跳过：N 个
```

---

## 注意事项

- 每次处理完一批 PDF 页面后立即删除临时文件
- 只建有公式/定量内容的 B/C 级卡，纯叙述章节跳过
- 不重复建已有卡片——先 `find` 再决定
- 同一概念多来源都涉及 → 只建一张卡 + `## 各来源应用`
- Paper 模式：通用概念（LCA、GWR 等）只引用不新建，只有原创方法才建卡
