---
name: install-agent-team
description: 将 agent-team 的 Java 企业级开发技能安装到目标项目。一次性安装程序。
---

# Agent Team 安装程序

将黄金四角技能（`/plan-java`、`/tdd-java`、`/guardrails-java`、`/code-review-java`）安装到目标 Java 项目。

---

## 安装流程

### 步骤 1：验证目标项目

确认当前目录是 Java 项目：

- 存在 `pom.xml` 或 `build.gradle` → 继续
- 不存在 → 提示"当前目录不是 Java 项目。请在 Java/Spring Boot 项目根目录运行此命令。"

### 步骤 2：检测已安装内容

```
检查目标项目：
  .agents/skills/plan-java/SKILL.md      → 已安装/未安装
  .agents/skills/tdd-java/SKILL.md       → 已安装/未安装
  .agents/skills/guardrails-java/SKILL.md → 已安装/未安装
  .agents/skills/code-review-java/SKILL.md → 已安装/未安装
  CLAUDE.md 中有 Agent skills 区块       → 已配置/未配置
  docs/agents/issue-tracker.md           → 已配置/未配置
```

### 步骤 3：选择安装范围

向用户展示：

```
发现已安装：[列出已存在的]
未安装：[列出缺失的]

请选择安装范围：
  1. 全部安装（覆盖已有文件）
  2. 只安装缺失的（跳过已有）
  3. 自定义选择
```

### 步骤 4：从 agent-team 源仓库复制文件

找到 agent-team 的安装路径，将技能文件复制到目标项目。复制规则：

| 源文件 | 目标路径 |
|---|---|
| `{agent-team}/.agents/skills/plan-java/SKILL.md` | `{target}/.agents/skills/plan-java/SKILL.md` |
| `{agent-team}/.agents/skills/tdd-java/SKILL.md` | `{target}/.agents/skills/tdd-java/SKILL.md` |
| `{agent-team}/.agents/skills/guardrails-java/SKILL.md` | `{target}/.agents/skills/guardrails-java/SKILL.md` |
| `{agent-team}/.agents/skills/code-review-java/SKILL.md` | `{target}/.agents/skills/code-review-java/SKILL.md` |
| `{agent-team}/docs/agents/issue-tracker.md` | `{target}/docs/agents/issue-tracker.md` |
| `{agent-team}/docs/agents/triage-labels.md` | `{target}/docs/agents/triage-labels.md` |
| `{agent-team}/docs/agents/domain.md` | `{target}/docs/agents/domain.md` |

**定位 agent-team 路径**：
1. 先检查当前目录是否就是 agent-team 本身（存在 `.agents/skills/plan-java/`）→ 源路径 = `.`
2. 否则在 `~/.claude/`、`~/projects/`、`D:\project\AIproject\` 等常见位置搜索 `agent-team` 目录
3. 找不到则询问用户输入 agent-team 仓库的路径

### 步骤 5：合并 CLAUDE.md

如果目标项目已有 `CLAUDE.md`：
- 检查是否已有 `## Agent skills` 区块
- 有 → 更新内容
- 没有 → 追加到文件末尾

如果目标项目有 `AGENTS.md` 而没有 `CLAUDE.md`：
- 创建 `CLAUDE.md`（内容从 AGENTS.md 合并）

如果两者都没有：
- 创建 `CLAUDE.md`，包含 Agent skills 配置区块

CLAUDE.md 中追加的内容（根据目标项目调整）：

```markdown
## Agent skills

### Issue tracker

GitHub Issues，使用 `gh` CLI 操作。外部 PR 不作为 triage 需求入口。详见 `docs/agents/issue-tracker.md`。

### Triage labels

使用默认标签：`needs-triage`、`needs-info`、`ready-for-agent`、`ready-for-human`、`wontfix`。详见 `docs/agents/triage-labels.md`。

### Domain docs

单一上下文布局 — `CONTEXT.md` + `docs/adr/` 位于仓库根目录。详见 `docs/agents/domain.md`。
```

### 步骤 6：验证安装

安装完成后检查：

- [ ] 四个技能文件已复制到 `.agents/skills/`
- [ ] `docs/agents/` 三个配置文件已就位
- [ ] `CLAUDE.md` 包含 Agent skills 区块
- [ ] `CONTEXT.md` 存在（如果不存在，提示运行 `/plan-java` 初始化）

### 步骤 7：汇报安装结果

```
✅ 安装完成

已安装技能：
  /plan-java          需求与架构师
  /tdd-java           TDD 研发一体机
  /code-review-java   双轴代码评审（Java 特化）
  /guardrails-java    工程护栏（按需运行一次）

已安装配置：
  docs/agents/issue-tracker.md
  docs/agents/triage-labels.md
  docs/agents/domain.md

下一步：
  1. 运行 /plan-java 初始化 CONTEXT.md 和版本基线
  2. 运行 /guardrails-java 安装 pre-commit 熔断 + CI/CD
```

---

## 安装后验证

在目标项目中，Agent 可运行：

```
/plan-java --dry-run    ← 验证技能可加载
```

---

## 注意事项

- 此安装程序本身也是 agent-team 仓库的一部分，只能从 agent-team 仓库运行
- 安装后的技能文件是独立副本，不会随 agent-team 仓库更新而自动更新
- 如需升级技能，重新运行安装程序并选择"全部安装（覆盖已有文件）"
