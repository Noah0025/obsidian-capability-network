# obsidian-capability-network

用 AI 把原始资料转成结构化能力网络的 Obsidian 工作流。

喂给它 PDF / 笔记 / 文档，它生成 wikilink 互联的概念卡 + 总结，写回你的 vault。

---

> 📖 **本 README**：写给人类——解释**这是什么 / 怎么组织的 / 设计原则**
> 🤖 **[QUICKSTART.md](QUICKSTART.md)**：写给 AI 代理——把仓库交给你的 AI 工具，让它自动完成安装、配置和首次摄入

---

## 这是什么

一套在 Obsidian 里运行的知识管理系统，核心思路：

**原始资料 → 喂给 AI → AI 整理成结构化知识卡 → Obsidian 网络**

不需要手动整理笔记。你只需要决定"这个东西值得吸收"，AI 负责拆解和组织。

兼容任何能读写本地文件的 AI 工具：Claude Code、Cursor、Cline 等。

---

## 知识层级

系统用 6 个层级描述知识，从宏观到原子：

```
Overview    全局地图（一张知识/能力网络的俯视图）
└── Field       领域地图（某个主题或方向）
    └── Summary     来源总结（一份资料 — 论文 / 报告 / 会议 / 课程 / 任意 source_type）
        └── Concept/A   视角卡（串联多个机制的主题）
            └── Concept/B   机制卡（完整知识单元，5-15 分钟能读完）
                └── Concept/C   原子卡（单一公式 / 规则 / 断言）
```

每一级都有对应的模板，在 `templates/` 目录下。

---

## 文件结构

```
obsidian-capability-network/
├── templates/
│   ├── overview.md       # Level 1 — 全局地图
│   ├── field.md          # Level 2 — 领域地图
│   ├── summary.md        # Level 3 — 来源总结
│   ├── concept_a.md      # Level 4 — 主题卡
│   ├── concept_b.md      # Level 5 — 机制卡
│   ├── concept_c.md      # Level 6 — 原子卡
│   ├── log.md            # 摄入日志（按时间，自动维护）
│   └── index.md          # 知识地图（按主题，自动维护）
├── skills/
│   ├── obsidian-ingest/           # 摄入器
│   ├── obsidian-audit/            # 健康检查
│   ├── obsidian-field-update/     # Field 同步
│   ├── obsidian-overview-update/  # Overview 同步
│   └── obsidian-workflow/         # 编排器（串联前 4 个）
├── QUICKSTART.md         # 约 10 分钟跑通指南（首次摄入 PDF 视长度多 3-5 分钟）
├── vault-config.example.md  # vault 路径/命名配置模板（按 AI 工具放到对应位置）
└── README.md
```

## Vault 安装后结构

```
your-vault/
├── Concepts/
├── Fields/
├── Overviews/
├── Sources/        # 子目录由 ingest 按 source_type 自动创建
├── Templates/
├── log.md          # 摄入日志，首次摄入时自动创建
└── index.md        # 知识地图，首次摄入时自动创建
```

---

## 工作流（9 步）

```
01-02  判断资料类型 + 提取文档内容
03     检查是否已有相关概念卡（新建 / 补充 / 跳过）
04     生成概念卡（A 主题 / B 机制 / C 原子）
05     生成总结文档（统一进 Sources/<source_type>/）
06     更新 log.md（按时间）+ index.md（按主题）
07     更新 Field 文档
08     更新 Overview 文档
09     周期性健康检查（断链 / 孤立卡 / 结构不合规）
```

入口指令（装好后）：

```
/obsidian-workflow /path/to/your/material
```

---

## 开始使用

把这个仓库交给你的 AI 工具（Claude Code / Cursor / Cline / Codex / 其他），然后说：

> "Read QUICKSTART.md and set this up for me."

AI 会引导你完成：
1. 检查依赖（poppler / OCR 工具）
2. 询问你的 vault 路径
3. 创建必要的目录结构
4. 安装模板和 skill
5. 按你的 AI 工具放置配置文件
6. 跑一次测试摄入验证

整个过程你只需要回答 1-2 个问题，剩下的 AI 自己处理。

---

## 模板说明

所有模板用 `{{placeholder}}` 标记需要填写的字段，`[[wikilink]]` 表示 Obsidian 内部链接。

| 模板 | 适用场景 |
|------|---------|
| `overview.md` | 整张知识地图的俯视图 |
| `field.md` | 某个领域或方向 |
| `summary.md` | 一份来源（课程 / 论文 / 项目 / 实习 / 毕业论文 / 其他） |
| `concept_a.md` | 视角卡：串联多个机制的主题 |
| `concept_b.md` | 知识单元：完整机制 / 方法 |
| `concept_c.md` | 原子卡：单个公式 / 规则 / 断言 |
| `log.md` | 摄入日志（按时间，自动维护，不需手填） |
| `index.md` | 知识地图（按主题/source_type 分组，自动维护，作为大 vault 的导航入口） |

---

## 命名规范

单词用 `_` 连接，默认英文，不含空格和特殊字符。

| 层级 | 格式 | 示例 |
|------|------|------|
| Overview | `Overview_<Name>` | `Overview_Research` |
| Field | `Field_<Name>` | `Field_Statistics` |
| Summary | `<Type>_<Name>_<YYYY>` | `Paper_Smith2024_Keyword` |
| Concept/A | `A_<Name>` | `A_Neural_Networks` |
| Concept/B | `B_<Name>` | `B_Backpropagation` |
| Concept/C | `C_<Name>` | `C_Chain_Rule` |

Summary 的 `<Type>` 前缀：

| 来源 | 前缀 | 示例 |
|------|------|------|
| 论文 | `Paper` | `Paper_Smith2024_Keyword` |
| 其他 | 任意前缀或自定义 | `Report_Q1_2026`、`Meeting_Kickoff`、`0217_Course_Name` |

---

## 设计原则

- **喂进去，不是整理进去**：原始资料不需要预处理，AI 负责理解和拆解
- **链接重于层级**：`[[wikilink]]` 让知识网络化，不只是树状结构
- **只建有价值的卡**：纯叙述、常识、操作步骤不建卡，降低维护负担
- **同一概念一张卡**：多个来源涉及同一概念，在一张卡里记录不同视角，不重复建卡
- **source_type 是开放字段**：学生用 paper/course/project，团队用 report/meeting/customer_call，工具不预设，按你的场景自由扩展

---

## 依赖（详细安装见 QUICKSTART）

- [Obsidian](https://obsidian.md/)
- 一个能读写本地文件的 AI 工具，任选其一：Claude Code / Cursor / Cline / Codex 等
- `pdftotext` + `pdftoppm`（PDF 处理：macOS `brew install poppler`，Linux `apt install poppler-utils`，Windows `choco install poppler` / `scoop install poppler` / 下载 https://github.com/oschwartz10612/poppler-windows）
- 可选 OCR 工具：`tesseract`（处理扫描件）

Windows 推荐使用 WSL 跑全流程。

---

## 安装

**给 AI agent**：把 [QUICKSTART.md](./QUICKSTART.md) 整份喂给你的 AI 代理。

**给人类**：跟 QUICKSTART 一步步来，约 10 分钟（首次摄入 PDF 视长度多 3-5 分钟）。

---

## License

MIT
