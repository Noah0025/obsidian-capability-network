---
name: obsidian-ingest
description: 从一个路径摄入原始资料（PDF / 课件 / 笔记），自动判断 source_type，生成对应概念卡 + 总结文档，更新 log.md 和 index.md。用法：/obsidian-ingest <路径> [--type=<source_type>]
---

# Obsidian Ingest（材料摄入）

输入一个路径，自动判断资料类型，生成对应的概念卡和总结文档。完成后输出摄入报告。

---

## CONFIG（使用前修改）

```
VAULT      = /path/to/your/obsidian/vault
CONCEPTS   = $VAULT/Concepts          # 概念卡根目录
SOURCES    = $VAULT/Sources           # 总结文档目录（默认按 source_type 自动分子目录）
FIELDS     = $VAULT/Fields            # 领域文档（供 ingest 引用 wikilink）
OVERVIEWS  = $VAULT/Overviews         # 全局视图（供 ingest 引用 wikilink）
TEMPLATES  = $VAULT/Templates         # 模板目录
LOG        = $VAULT/log.md            # 摄入日志（自动维护）
INDEX      = $VAULT/index.md          # 知识地图（自动维护）
```

> 用户可在 `$SOURCES/` 下任意建子目录（如 `paper/`、`report/`、`meeting/`、`customer_call/`，小写下划线），按个人/团队习惯组织。本 skill 不预设子目录。

> Tip: 可将此 CONFIG 块复制到 `CLAUDE.md`，让 Claude 自动读取，无需每次手动指定。

---

## Step 0 — 类型判断与目标确认

1. 输入 `<path>` 为路径。若不存在，提示并退出。
2. `ls` 盘点内容。
3. **判断 source_type**（最少两种自动识别，其余由用户指定）：

| 判断条件 | source_type | 命名建议 |
|---------|------|---------|
| 路径是单个 `.pdf` 文件，PDF 内能识别出 DOI / Abstract / 多作者结构 | **paper** | `Paper_<FirstAuthor><YYYY>_<Keyword>` |
| 文件夹内全是论文 PDF（无课件/多章节） | **paper**（逐篇处理） | 同上 |
| 其他情况 | 由用户在调用时指定 `--type=<任意名>`，或由 Claude 从内容推断（如 `course`/`report`/`meeting`/`customer_call` 等） | `<Type>_<Name>` 或用户自定义 |

> 不预设固定类型清单，`source_type` 是开放字段。用户既可以用常见词（course/paper/project），也可以根据自己场景定义新词（report/meeting/spec/customer_interview…）。

**输出目录**：统一放在 `$SOURCES/<source_type>/`（首次出现的 type 自动创建子目录）；用户也可以预先在 `$SOURCES/` 下手建好子目录，本 skill 会沿用现有结构。

**概念卡命名**（Claude 生成时遵循）：
- Concept/A：`A_<ConceptName>`，如 `A_Neural_Networks`
- Concept/B：`B_<ConceptName>`，如 `B_Backpropagation`
- Concept/C：`C_<ConceptName>`，如 `C_Chain_Rule`
- 单词用 `_` 连接，默认英文，首字母大写

4. 检查对应总结文档是否已存在 → 已存在则切换**补充模式**（只补概念卡和缺失章节）。

---

## Step 1 — 提取原始资料内容

### PDF 处理策略

**先尝试文字提取**：
```bash
pdftotext "file.pdf" - 2>/dev/null | head -100
echo "EXIT:$?"
```
退出码判断：
- 输出有实质内容 → 直接用
- 输出为空或乱码，退出码 0 → 扫描件，进入 OCR
- 退出码非 0，含 "encrypted"/"permission" → 标记加密，仅用文件名
- 文件不存在（退出码 1）→ 报错退出

**扫描件 OCR**（每次 10-15 页，处理完立即删临时文件）：
```bash
OCR_TMP=$(mktemp -d)
pdftoppm -r 150 -f <start> -l <end> "file.pdf" "$OCR_TMP/page"
for f in "$OCR_TMP"/page-*.ppm; do tesseract "$f" stdout 2>/dev/null; echo "---"; done
rm -rf "$OCR_TMP"
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

## Step 5 — 更新 Log 与 Index

### 5a. 追加 Log（按时间）

在 `$LOG`（默认 `$VAULT/log.md`）记录本次摄入，作为审计日志。

1. 若 `$LOG` 不存在 → 从 `$TEMPLATES/log.md` 复制创建（QUICKSTART 已默认拷过，本步是兜底）。
2. 读取 `$LOG`，检查是否有 `## YYYY-MM`（当月）段落：
   - **没有** → 在 `# Ingest Log` 标题之后、第一个现有 `## YYYY-MM` 之前插入新月份段落（含表头）。
   - **有** → 直接进入下一步。
3. 在当月表格末尾追加一行：

```
| <YYYY-MM-DD> | <资料名> | <类型> | +N (A:N B:N C:N) | +N | [[<总结文档名>]] |
```

字段定义：
- 资料名：原始文件名或文件夹名（不含路径）
- 类型：Step 0 判定的 `source_type`（开放字段，常见值 `paper`/`course`/`report`/`meeting`/...）
- 新建卡：本次 Step 3 新建的概念卡数量（按 A/B/C 分别计）
- 补充卡：本次 Step 2 补入「各来源应用」的卡数量
- 总结文档：本次 Step 4 生成的 wikilink

### 5b. 更新 Index（按主题）

在 `$INDEX`（默认 `$VAULT/index.md`）登记本次新增的总结文档，作为知识地图入口（当 vault 大到记不住时的导航）。

1. 若 `$INDEX` 不存在 → 从 `$TEMPLATES/index.md` 复制创建（QUICKSTART 已默认拷过，本步是兜底）。
2. 读取 `$INDEX`，定位 `## <source_type>` 章节：
   - **找到** → 在该章节末尾追加一行：`- [[<总结文档名>]] — <TL;DR 第一句>`
   - **未找到** → 在文档末尾插入新章节 `## <source_type>` + 同样一行
3. **补充模式**（Step 4 状态为"补充"）：若该 wikilink 已在 index 中，则不重复追加；若 TL;DR 有更新，替换原描述。

**TL;DR 第一句**：从本次生成的总结文档 `## TL;DR` 章节取第一个句号前的内容（最多 80 字符，超出截断加 "…"）。

**容错**：5a 或 5b 任一写入失败，记录到报告 `## 警告` 节并继续，不中断流程。

---

## Step 6 — 输出报告

报告首行必须输出结构化字段，供 obsidian-workflow 解析：

```
DOC_NAME: <总结文档名（不含.md）>
DOC_PATH: <总结文档完整路径>

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

确定命名：`Paper_FirstAuthorYear_Keyword`（如 `Paper_Smith2024_Keyword`）。

检查 `$SOURCES/paper/` 中是否已存在同名文件 → 已存在则切换补充模式。

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

参照 `templates/summary.md`（source_type: paper），放入 `$SOURCES/paper/`：

```markdown
---
tags:
  - Summary/Paper
source_type: paper
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

## 应用价值

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

## Step P4 — 更新 Log 与 Index

对每篇 Paper（批量时逐篇）执行与课程模式 Step 5 相同的 Log 追加 + Index 更新（见前文 5a/5b）。
`source_type` 字段填 `paper`。

---

## Step P5 — 输出报告

单篇时首行输出结构化字段；批量时每篇一组：

```
DOC_NAME: <Paper文档名（不含.md）>
DOC_PATH: <完整路径>
# 批量时重复上述两行，每篇一组

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
