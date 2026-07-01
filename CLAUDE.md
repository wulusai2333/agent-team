# Agent Team — Java 企业级开发 Agent 团队

基于 mattpocock/skills 设计哲学，针对 Java/Spring Boot 生态深度定制。

## 快速开始

### 第一步：筑基

在目标 Java 项目根目录创建 `CONTEXT.md`，写入当前项目的技术栈基线：

```markdown
# 项目全局业务与架构上下文

## 技术栈基线

| 组件 | 版本/方案 |
|---|---|
| JDK | 17 (Eclipse Temurin) |
| Spring Boot | 3.2.5 |
| Spring Cloud | 2023.0.3 |
| Spring Cloud Alibaba | 2023.0.1.0 |
| 数据库 | PostgreSQL 15 |
| 缓存 | Redis 7 (Redisson) |
| 消息队列 | RocketMQ 5.x |
```

### 第二步：装锁

在 Claude Code 中运行 `/guardrails-java`，一次性安装：
- `.git/hooks/pre-commit` — 本地静态红线拦截（毫秒级）
- `.github/workflows/ci.yml` — CI/CD 全量测试 + 安全扫描
- `Dockerfile` + `docker-compose-env.yml` — 容器化底座

### 第三步：开启研发流

新需求来了：
1. 运行 `/plan-java` → 三维盘问 + 版本校验 → 产出 ADR + PRD + 垂直切片 Issue
2. 运行 `/tdd-java 抓取并执行当前 Issue #[N]` → 红→绿→微重构→提交
3. 提交后 `/code-review` 自动评审 → CI 护栏自动扫描 → 合并

---

## Agent skills

### Issue tracker

GitHub Issues（https://github.com/wulusai2333/agent-team），使用 `gh` CLI 操作。外部 PR 不作为 triage 需求入口。详见 `docs/agents/issue-tracker.md`。

### Triage labels

使用默认标签：`needs-triage`、`needs-info`、`ready-for-agent`、`ready-for-human`、`wontfix`。详见 `docs/agents/triage-labels.md`。

### Domain docs

单一上下文布局 — `CONTEXT.md` + `docs/adr/` 位于仓库根目录。详见 `docs/agents/domain.md`。
