# learning-log 项目规则

## 这是什么

我的学习与求职准备记录（过去、现在、未来的学习），同时是一个 Obsidian vault。
GitHub 公开仓库：`jinyuanzhan/learning-log`。

## 公开与私有

- `CV/`、`interview/`、`journal/` 只保存在本地，已写进 `.gitignore`，不会推送到 GitHub。云端会话看不到这三个文件夹。
- 其他内容都是公开的。写进公开文件夹的内容不要带个人隐私、具体公司的面试细节、面试官信息；这类内容放 `interview/`。
- 不要用 `git add -f` 把这三个文件夹加回来。

## 在线目录

- HTML 笔记通过 GitHub Pages 在线阅读：`https://jinyuanzhan.github.io/learning-log/`。
- 新增、改名或删除 HTML 笔记时，同步更新根目录的 `index.html`（网站首页）和 `README.md` 里的目录。

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

- **专业术语一律用英文**，任何文件里都不要翻译，例如 homography、SVD、pose estimation。
- 其他内容（解释、正文、个人记录）可以用中文。
- **Commit message 一律用英文**，简洁、祈使句，例如 `Add notes on attention mechanism`。

## 格式

- 文件名：小写英文加连字符，例如 `binary-search.md`。
- 每个笔记开头有 YAML frontmatter，至少包含 `created: YYYY-MM-DD` 和 `tags: [...]`。
- 知识点笔记建议结构：Summary → Key Ideas → Example / Code → Interview Questions → Related。
- 笔记之间的链接用 Obsidian 的 `[[file-name]]`；图片放 `assets/`，用 `![[image.png]]` 引用。

## 工作方式

- 所有会话（本地、云端、项目子线程）：改完后直接提交并推送到 `main`，不开 PR。
- 只保留一条线性的 `main` 历史：不建分支，不产生 merge commit。
  1. 开始修改前先 `git pull --rebase`，拿到最新内容。
  2. 推送前再 `git pull --rebase`，把本地提交接到远程最新提交之后，然后 `git push`。
  3. rebase 出现冲突时，保留双方内容并解决冲突后 `git rebase --continue`；拿不准就 `git rebase --abort` 并问我。
- 不删除已有笔记，只修改或补充；需要删除时先问我。
