# study-os

把 Claude 当作知识整理引擎的 Obsidian 工作流。

喂给它原始资料（PDF、笔记、论文），它帮你建结构化的知识网络。

---

## 这是什么

一套在 Obsidian 里运行的知识管理系统，核心思路：

**原始资料 → 喂给 Claude → Claude 整理成结构化知识卡 → Obsidian 网络**

不需要手动整理笔记。你只需要决定"这个东西值得学"，Claude 负责拆解和组织。

---

## 知识层级

系统用 6 个层级描述知识，从宏观到原子：

```
Overview    全局知识地图（整个学科的俯视图）
└── Field       领域地图（某个专业方向）
    └── Summary     来源总结（一门课 / 一篇论文 / 一个项目）
        └── Concept/A   主题卡（多个机制的集合）
            └── Concept/B   机制卡（完整知识单元，5-15分钟能读完）
                └── Concept/C   原子卡（单个公式 / 规则）
```

每一级都有对应的模板，在 `templates/` 目录下。

---

## 文件结构

```
study-os/
├── templates/
│   ├── overview.md       # Level 1 — 全局地图
│   ├── field.md          # Level 2 — 领域地图
│   ├── summary.md        # Level 3 — 来源总结（课程/论文/项目/实习/论文）
│   ├── concept_a.md      # Level 4 — 主题卡
│   ├── concept_b.md      # Level 5 — 机制卡
│   └── concept_c.md      # Level 6 — 原子卡
├── skills/
│   └── obsidian-ingest/
│       └── SKILL.md      # Claude Code skill：自动摄入原始资料
└── README.md
```

---

## 快速开始

### 1. 配置路径

把你的 Obsidian vault 路径填入 `skills/obsidian-ingest/SKILL.md` 顶部的 CONFIG 块：

```
VAULT      = /path/to/your/vault
CONCEPTS   = $VAULT/Concepts
COURSES    = $VAULT/Courses
...
```

或者直接写进你项目的 `CLAUDE.md`，Claude 会自动读取。

### 2. 安装模板

把 `templates/` 里的文件复制到你 Obsidian vault 的 Templates 文件夹。

### 3. 摄入资料

在 Claude Code 里运行：

```
/obsidian-ingest /path/to/your/materials
```

Claude 会：
- 判断资料类型（课程 / 论文 / 项目 / 实习 / 毕业论文）
- 提取内容，生成概念卡（B/C 级）
- 生成总结文档，链接相关概念
- 输出摄入报告

---

## 模板说明

所有模板用 `{{placeholder}}` 标记需要填写的字段，`[[wikilink]]` 表示 Obsidian 内部链接。

| 模板 | 适用场景 |
|------|---------|
| `overview.md` | 整个学科或专业的俯视图 |
| `field.md` | 某个专业方向（如 Machine Learning、Hydrology） |
| `summary.md` | 一篇论文 / 一个项目 / 实习 / 毕业论文 |
| `concept_a.md` | 多个机制的集合（如 Neural Networks） |
| `concept_b.md` | 完整机制（如 Backpropagation） |
| `concept_c.md` | 单个公式或规则（如 Chain Rule） |

---

## 命名规范

单词用 `_` 连接，默认英文，不含空格和特殊字符。

| 层级 | 格式 | 示例 |
|------|------|------|
| Overview | `Overview_<Name>` | `Overview_Computer_Science` |
| Field | `Field_<Name>` | `Field_Machine_Learning` |
| Summary | `<Type>_<Name>_<YYYY>` | `Paper_Smith2024_UrbanFlood` |
| Concept/A | `A_<Name>` | `A_Neural_Networks` |
| Concept/B | `B_<Name>` | `B_Backpropagation` |
| Concept/C | `C_<Name>` | `C_Chain_Rule` |

Summary 的 `<Type>` 前缀：

| 来源 | 前缀 | 示例 |
|------|------|------|
| 论文 | `Paper` | `Paper_Smith2024_UrbanFlood` |
| 毕业论文 | `Thesis` | `Thesis_MSc_2025` |
| 项目 | `Proj` | `Proj_Water_Quality` |
| 实习 | `Intern` | `Intern_KWB_2024` |

---

## 设计原则

- **喂进去，不是整理进去**：原始资料不需要预处理，Claude 负责理解和拆解
- **链接重于层级**：`[[wikilink]]` 让知识网络化，不只是树状结构
- **只建有价值的卡**：纯叙述、常识、操作步骤不建卡，降低维护负担
- **同一概念一张卡**：多门课涉及同一概念，在一张卡里记录不同视角，不重复建卡

---

## 依赖

**必须**
- [Obsidian](https://obsidian.md)
- [Claude Code](https://claude.ai/code)（运行 skill 需要）
- `pdftotext` + `pdftoppm`：PDF 处理
  ```bash
  brew install poppler        # macOS
  sudo apt install poppler-utils  # Linux
  ```

**可选（扫描件 OCR）**
- 任意 OCR 命令行工具，放到 PATH 下（`tesseract` 推荐）：
  ```bash
  brew install tesseract      # macOS
  sudo apt install tesseract-ocr  # Linux
  ```
  在 skill CONFIG 中将 `ocr_tool` 指向你的 OCR 命令。
