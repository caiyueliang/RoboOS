# Issue tracker: GitHub

本仓库的 Issues 和 PRD 都作为 GitHub issues 存在。所有操作都使用 `gh` CLI。

## Conventions

- **创建 issue**: `gh issue create --title "..." --body "..."`。多行内容使用 heredoc。
- **读取 issue**: `gh issue view <number> --comments`，使用 `jq` 过滤评论并获取 labels。
- **列出 issues**: `gh issue list --state open --json number,title,body,labels,comments --jq '[.[] | {number, title, body, labels: [.labels[].name], comments: [.comments[].body]}]'`，配合 `--label` 和 `--state` 过滤器使用。
- **评论 issue**: `gh issue comment <number> --body "..."`
- **添加/移除 labels**: `gh issue edit <number> --add-label "..."` / `--remove-label "..."`
- **关闭**: `gh issue close <number> --comment "..."`

从 `git remote -v` 推断仓库 — `gh` 在 clone 内运行时会自动完成。

## When a skill says "publish to the issue tracker"

创建一个 GitHub issue。

## When a skill says "fetch the relevant ticket"

运行 `gh issue view <number> --comments`。
