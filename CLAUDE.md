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
| `mvn -v` 或 `./gradlew -v` | `/tdd-java`、`/guardrails-java` | 构建与测试 |
| `gh --version` | `/plan-java` | Issue 发布 |
| `docker --version` | `/tdd-java`（Testcontainers）、`/guardrails-java` | 中间件容器 |

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

## 构建故障恢复指南

本章是所有 Skill 的共享参考。当 Maven/Gradle 构建失败时，按此优先级排查。

### 故障分类与恢复策略

```
构建失败
  ├── 编译错误（cannot find symbol / compilation error）
  │     → 代码问题，不是环境问题
  │     → Agent 自行修复：检查 import、类名拼写、依赖是否添加到 POM
  │
  ├── 依赖下载失败（Could not resolve dependencies）
  │     ├── 网络问题 → 提示用户检查网络
  │     ├── 公司私有仓库不可达 → 检查 ~/.m2/settings.xml 的 mirror 配置
  │     └── 镜像源问题 → 建议在 pom.xml 中配置阿里云 mirror
  │
  ├── JDK 版本不匹配（class file has wrong version）
  │     → 检查 JAVA_HOME 和 pom.xml 中的 <java.version>
  │     → 提示用户：当前 JAVA_HOME 指向哪个版本、项目需要哪个版本
  │
  ├── 本地仓库损坏（Could not find artifact / corrupted jar）
  │     → 删除 ~/.m2/repository 中对应依赖的目录，重新下载
  │     → 命令：rm -rf ~/.m2/repository/{groupId}/{artifactId}
  │
  ├── 端口被占用（Port xxx already in use）
  │     → 查找占用进程：netstat -ano | findstr :{port}
  │     → 杀掉进程或更换端口
  │
  ├── Docker 容器问题
  │     ├── 容器未启动 → docker-compose -f docker-compose-env.yml up -d
  │     ├── 端口冲突 → 修改 docker-compose-env.yml 中的 ports 映射
  │     ├── 镜像拉取失败 → 配置镜像加速器或使用本地已有镜像
  │     └── 磁盘空间不足 → docker system prune -a
  │
  └── 内存不足（Java heap space / GC overhead）
        → 减少并行测试线程数：mvn test -T 1
        → 增加 Maven 堆内存：export MAVEN_OPTS=-Xmx1024m
```

### Agent 行为规则

**最大自治原则**：
- 编译错误 → 自行修复，最多重试 3 次
- 依赖下载失败 → 重试 1 次（可能只是网络抖动），仍失败则报告用户
- JDK 版本问题 → 不自行切换 JDK，报告用户当前环境与项目要求的差异
- Docker 问题 → 不自行重启 Docker Desktop，报告用户
- 同一错误重试 3 次仍失败 → 不再重试，详细报告用户排查步骤

**报告模板**（当需要用户介入时）：

```
⚠️ [BUILD FAILED] {错误类型}

原因：{一句话}

当前环境：
  Java: {版本}（JAVA_HOME: {路径}）
  Maven: {版本}
  Docker: {可用/不可用}

建议修复：
  1. {具体步骤}
  2. {备选方案}

如需要我代为执行，请回复"授权"。
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
