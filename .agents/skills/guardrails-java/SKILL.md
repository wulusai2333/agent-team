---
name: guardrails-java
description: Java 工程护栏一次性安装程序。生成 pre-commit 熔断脚本（Bash + PowerShell）、OWASP 安全扫描、Springdoc 文档自动生成、Docker 容器化、GitHub Actions CI/CD、ELK 配置、K8s 清单。安装后护栏沉默运行。
---

# 哲学定位

**一次性安装程序**，不是对话式 Agent。运行时生成所有配置，然后退出。护栏本身通过 Git hook、Maven 插件、GitHub Actions **静默自动执行**，触碰红线时熔断拦截。

---

# 断点续传

重新唤醒时扫描已有配置（pre-commit / OWASP / Springdoc / Dockerfile / docker-compose / CI / K8s / ELK），跳过已生成模块，仅安装缺失部分。详见 `REFERENCE.md#断点续传`。

---

# 安装模块

| 模块 | 产出 | 触发 |
|---|---|---|
| 1. Pre-commit | `.git/hooks/pre-commit` + `.git/hooks/pre-commit.ps1` | 本地 git commit |
| 2. 安全扫描 | `pom.xml` 中 OWASP + CI 中 Trivy | CI `mvn verify` / Docker 构建 |
| 3. API 文档 | Springdoc 依赖 + 配置 | `mvn compile` |
| 4. Docker | `Dockerfile` + `docker-compose-env.yml` | 本地开发 / CI 构建 |
| 5. CI/CD | `.github/workflows/ci.yml` | git push |
| 6. K8s | `k8s/deployment.yaml` + StatefulSet | 按需 |
| 7. ELK | Logstash pipeline + Filebeat DaemonSet + ES 模板 | 按需 |

**动静分离**：本地 pre-commit 仅 `mvn compile -q` + grep 静态扫描（秒级）。全量测试 + OWASP 全在 CI 远端执行。

---

# 交互约束

| 步骤 | 模板 |
|---|---|
| 项目检测 | `检测到：{Maven/Gradle}，Java {版本}，Boot {版本}。` |
| 各模块 | `{模块名} 已生成/已跳过。` |
| 安装完成 | 激活清单表格 + 下一步指引 |

---

# 完成后交接

```
护栏安装完成。已激活：
  本地：.git/hooks/pre-commit（静态红线，秒级）
  远端：.github/workflows/ci.yml（全量测试 + OWASP + Trivy）

下一步：
  git add .git/hooks/ .github/ && git commit -m "chore: 安装工程护栏"
  /plan-java 开始第一个需求
```

所有模板脚本、完整 YAML/XML 配置、反模式清单、回退策略详见 `REFERENCE.md`。
