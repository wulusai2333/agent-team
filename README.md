# Agent Team — Java 企业级开发 Agent 团队

基于 [mattpocock/skills](https://github.com/mattpocock/skills) 设计哲学，针对 Java/Spring Boot 生态深度定制的 Claude Code Agent 技能组。

## 是什么

六个 Claude Code 技能，覆盖从需求到部署的完整软件交付链路：

```
/plan-java           需求与架构师 → 盘问 + 版本大闸 → ADR + PRD + Issue
/tdd-java            TDD 研发一体机 → 红→绿→微重构→提交
/code-review-java    双轴代码评审 → 规范 + 业务对照
/guardrails-java     工程护栏 → pre-commit + CI/CD + Docker + ELK + K8s（一次性安装）
/install-agent-team  安装程序 → 将本技能组安装到其他项目
```

## 安装

### 方式一：安装到当前项目

1. 克隆本仓库到本地
2. 在 **目标 Java 项目根目录** 的 Claude Code 中运行 `/install-agent-team`
3. Agent 会自动检测缺失的技能并安装

### 方式二：手动复制

将以下目录复制到目标项目的 `.agents/skills/` 下：

```
plan-java/SKILL.md
tdd-java/SKILL.md
code-review-java/SKILL.md
guardrails-java/SKILL.md
```

---

## 使用

### 前置条件

| 工具 | 用途 | 必须？ |
|---|---|---|
| JDK 17+ | 编译运行 | ✅ |
| Maven 3.9+ 或 Gradle | 构建 | ✅ |
| Git | 版本管理 | ✅ |
| Docker Desktop | 本地中间件 | ⚠️ 没有也能用，自动退化到 H2/内嵌模式 |
| GitHub CLI (`gh`) | Issue 管理 | ⚠️ `/plan-java` 需要 |

### 三步上手

**第一步：筑基**

在项目根目录创建 `CONTEXT.md`，写入技术栈基线。如果项目已有技术栈，这一步可以跳过——`/plan-java` 会在第一次运行时自动创建。

**第二步：装锁**

在 Claude Code 中运行：

```
/guardrails-java
```

Agent 会一次性生成：
- 本地 pre-commit 熔断脚本（拦截 `javax.*` 污染、缺少 CONTEXT.md）
- GitHub Actions CI/CD 流水线（编译 + 测试 + OWASP 安全扫描 + Trivy 镜像扫描）
- Dockerfile（多阶段构建）+ docker-compose-env.yml（中间件底座）
- K8s 部署清单（Deployment + Service + StatefulSet）

**第三步：开发**

来了新需求：

```
/plan-java
```

Agent 会盘问业务需求、校验版本兼容性、生成 ADR + PRD + 垂直切片 Issue。然后：

```
/tdd-java 抓取并执行当前 Issue #1
```

Agent 进入红→绿→微重构→提交的 TDD 循环。提交后：

```
/code-review-java
```

双轴评审（编码规范 + 业务 AC 对照）。

---

## 技能详解

### /plan-java — 需求与架构师

```
输入：一句话需求
输出：CONTEXT.md + ADR + PRD + GitHub Issues（带 ready-for-agent 标签）
```

行为：
1. 三维盘问（业务流、技术基座、性能一致性）
2. 版本兼容性校验（Jakarta 命名空间、Spring 三位一体、中间件驱动）
3. 数据一致性审查（Cache-Aside 默认、事务消息 vs Seata）
4. 生成 ADR（仅当满足"难以逆转 + 令人困惑 + 真实权衡"三条件）
5. 生成 PRD + 垂直切片 Issue

**变化触发机制**：如果 CONTEXT.md 中已有技术栈基线且不涉及升级，自动跳过全量版本审计，避免 Token 浪费。

### /tdd-java — TDD 研发一体机

```
输入：Issue
输出：测试代码 + 实现代码 + Git 提交
```

循环：红灯（含编译失败）→ 绿灯 → 微重构 → 提交

Java 特化：
- 红灯-1（编译失败）是合法的，先写骨架再进断言
- 多模块项目自动用 `-pl :module-name` 避免扫描全量模块
- Testcontainers 不可用时自动回退到 H2 + embedded-redis
- 微重构只整理刚写的代码，宏观重构留给 review

### /code-review-java — 双轴代码评审

```
输入：基准点（commit/branch/tag）
输出：Standards 报告 + Spec 报告
```

两个轴并行：
- **Standards**：编码规范 + Spring 分层反模式 + Fowler 坏味道基线
- **Spec**：对照 Issue/PRD 验收标准，检查缺失、范围蔓延、实现错误

### /guardrails-java — 工程护栏

```
输入：无（一次性安装）
输出：pre-commit + CI.yml + Dockerfile + docker-compose-env.yml + K8s manifests + ELK 配置
```

- **本地 pre-commit**：秒级静态检查（javax.* 污染、CONTEXT.md 缺失），**不跑全量测试**
- **CI/CD**：全量测试 + OWASP CVE 扫描 + Trivy 镜像扫描
- **Docker**：多阶段构建 Dockerfile + 中间件 docker-compose-env.yml
- **ELK**：Logstash pipeline + Filebeat K8s DaemonSet + ES 索引模板
- **K8s**：Deployment + Service + StatefulSet（Nacos/RocketMQ）

---

## 环境退化机制

CLAUDE.md 步骤 0 在每次会话启动时探测环境，按结果降级：

| 环境 | 效果 |
|---|---|
| Java + Maven + Docker 全有 | 全部功能正常 |
| 缺 Docker | TDD 回退到 H2/内嵌，guardrails 跳过 Docker 模块 |
| 完全空白（新装 Windows） | 熔断所有技能，启动 winget 安装引导 |

---

## 项目结构

```
agent-team/
├── README.md                               ← 你在看
├── CLAUDE.md                               ← 环境大闸 + 故障恢复 + Agent skills 配置
├── docs/agents/
│   ├── issue-tracker.md                    ← GitHub Issues 操作约定
│   ├── triage-labels.md                    ← Triage 标签映射
│   └── domain.md                           ← 领域文档消费规则
└── .agents/skills/
    ├── plan-java/SKILL.md                  ← 需求与架构师
    ├── tdd-java/SKILL.md                   ← TDD 研发一体机
    ├── code-review-java/SKILL.md           ← 双轴代码评审
    ├── guardrails-java/SKILL.md            ← 工程护栏
    └── install-agent-team/SKILL.md         ← 安装程序
```

## 常见问题

**Q: 为什么 /guardrails-java 没加 `/`？**
加了的。四个用户命令统一为 `/plan-java`、`/tdd-java`、`/code-review-java`、`/guardrails-java`。

**Q: 能在非 GitHub 项目中使用吗？**
Issue Tracker 部分需要适配。目前 `/plan-java` 默认用 `gh` CLI 创建 Issue。如果你用 GitLab 或本地 Markdown，需要修改 `docs/agents/issue-tracker.md`。

**Q: 这套技能和原版 mattpocock/skills 的关系？**
基于原版设计哲学重写，不是 fork。核心改动：
- TypeScript → Java/Spring Boot 生态
- 增加了版本兼容性大闸
- 增加了多模块 Maven 支持
- 增加了金融系统幂等/审计专项检查
- 增加了环境退化机制

**Q: 能在 IntelliJ IDEA 中使用吗？**
Claude Code 是终端工具，不依赖 IDE。在 VS Code 终端、IntelliJ 终端、Windows Terminal 中都可以用。
