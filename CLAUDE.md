# learning-log 项目规则

## 这是什么

我的学习与求职准备记录（过去、现在、未来的学习），同时是一个 Obsidian vault。
GitHub 私有仓库：`jinyuanzhan/learning-log`。

## 内容放哪

| 内容 | 位置 |
|---|---|
| 知识点笔记 | `notes/<topic>/`：`algorithms`、`cs-basics`、`ml-llm`、`robotics`，一个知识点一个文件 |
| 学习日志 | `journal/YYYY-MM.md`，追加到当天日期 `### YYYY-MM-DD` 下 |
| 学习路线 | `roadmap.md`：已完成 / 进行中 / 未来要学 |
| 面试题、自我介绍、面试复盘 | `interview/`，复盘放 `interview/reviews/` |
| 项目总结 | `projects/`，一个项目一个文件 |
| 简历、Cover letter | `CV/` |
| 还没想清楚的想法 | `thinking/`，frontmatter 标 `status: thinking` 或 `status: done` |
| 图片等附件 | `assets/` |

没有合适的主题文件夹时，先问我，不要自己新建顶层文件夹。

## 语言

- **专业内容用英文**：`notes/`、`interview/`、`projects/`、`CV/` 里的技术内容、术语、定义、代码注释都用英文。
- 个人记录（`journal/`、`thinking/`）可以用中文，但技术术语保持英文，不要翻译。
- **Commit message 一律用英文**，简洁、祈使句，例如 `Add notes on attention mechanism`。

## 格式

- 文件名：小写英文加连字符，例如 `binary-search.md`。
- 每个笔记开头有 YAML frontmatter，至少包含 `created: YYYY-MM-DD` 和 `tags: [...]`。
- 知识点笔记建议结构：Summary → Key Ideas → Example / Code → Interview Questions → Related。
- 笔记之间的链接用 Obsidian 的 `[[file-name]]`；图片放 `assets/`，用 `![[image.png]]` 引用。

## 工作方式

- 本地会话：改完后提交并推送到 `main`。
- 云端会话 / 项目子线程：在新分支上改，完成后开 PR，不直接推 `main`。
- 不删除已有笔记，只修改或补充；需要删除时先问我。
