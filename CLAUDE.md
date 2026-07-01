# Agent Team — Java 企业级开发 Agent 团队

基于 mattpocock/skills 设计哲学，针对 Java/Spring Boot 生态深度定制。

---

## [CRITICAL] 步骤 0：环境就绪大闸（宿主机探针）

**本规则高于所有 Skill。** 在执行任何以 `/` 开头的自定义 Skill（`/plan-java`、`/tdd-java`、`/guardrails-java`）之前，必须先隐式探测宿主机环境。探测成功与否决定后续行为的退化策略。

### 环境探测清单

在后台静默执行（不向用户展示原始输出）：

| 检测命令 | 关联 Skill | 用途 |
|---|---|---|
| `git --version` | 全部 | 版本管理、Hook 安装 |
| `java -version` | `/plan-java`、`/tdd-java` | 编译与运行 |
| `mvn -v` 或 `./gradlew -v` | `/tdd-java`、`guardrails` | 构建与测试 |
| `gh --version` | `/plan-java` | Issue 发布 |
| `docker --version` | `/tdd-java`（Testcontainers）、`guardrails` | 中间件容器 |

### 环境缺失退化策略

```
探测结果：
  ├── 完全空白（java 不可达）
  │     → 熔断所有 Skill
  │     → 启动【环境引导向导】：输出 winget 一键安装指令
  │     → 等待用户确认环境就绪后继续
  │
  ├── 有 Java + Maven，但无 Docker
  │     → /plan-java 正常执行（只做架构设计，不需要 Docker）
  │     → /tdd-java 强制退化：Testcontainers 禁用，所有数据层测试回退到 H2 / 内嵌 Redis
  │     → /guardrails-java 跳过 Docker 相关模块
  │
  ├── 有 Java + Maven + Docker
  │     → 全部 Skill 正常运行
  │     → /tdd-java 可用 Testcontainers（需先验证 Docker Daemon 可访问）
  │
  └── 全部就绪
        → 全部 Skill 正常运行
```

### 白板 Windows 11 环境引导向导

当探测到完全空白环境时，输出以下引导（不执行，等待用户授权）：

```
⚠️ [CRITICAL ENV MISSING] 当前环境缺少 Java 开发基础设施。

推荐安装（按顺序）：

1. JDK 17 LTS (Eclipse Temurin)
   winget install EclipseAdoptium.Temurin.17.JDK

2. Git
   winget install Git.Git

3. Maven
   winget install Apache.Maven.3

4. Docker Desktop（本地测试中间件底座）
   winget install Docker.DockerDesktop

5. GitHub CLI（Issue 管理）
   winget install GitHub.cli

安装完成后，重新打开终端验证：
   java -version && mvn -v && git --version && docker --version

回复"授权安装"我将尝试自动执行以上 winget 命令。
```

---

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
