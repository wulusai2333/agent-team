# Issue Tracker：GitHub

本仓库的 Issue 和 PRD 以 GitHub Issues 形式管理，所有操作使用 `gh` CLI。

## 操作约定

- **创建 Issue**：`gh issue create --title "..." --body "..."`。多行正文使用 heredoc。
- **查看 Issue**：`gh issue view <number> --comments`，用 `jq` 过滤评论，同时获取标签。
- **列出 Issue**：`gh issue list --state open --json number,title,body,labels,comments --jq '[.[] | {number, title, body, labels: [.labels[].name], comments: [.comments[].body]}]'`，配合 `--label` 和 `--state` 筛选。
- **评论 Issue**：`gh issue comment <number> --body "..."`
- **添加/移除标签**：`gh issue edit <number> --add-label "..."` / `--remove-label "..."`
- **关闭 Issue**：`gh issue close <number> --comment "..."`

在仓库内执行时，`gh` 会自动根据 `git remote -v` 推断仓库信息。

## PR 作为需求入口

**PR 作为需求入口：否。**

## 当技能说"发布到 Issue Tracker"

创建一个 GitHub Issue。

## 当技能说"获取相关工单"

执行 `gh issue view <number> --comments`。
