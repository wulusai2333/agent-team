# guardrails-java 参考配置与脚本

按需读取。Agent 在执行到具体模块时读取对应章节，不全部加载。

---

# 断点续传

```
1. .git/hooks/pre-commit → 跳过模块 1
2. pom.xml 中有 owasp:dependency-check-maven → 跳过模块 2
3. pom.xml 中有 springdoc-openapi → 跳过模块 3
4. Dockerfile → 跳过 4.1
5. docker-compose-env.yml → 跳过 4.2
6. .github/workflows/ci.yml → 跳过模块 5
7. k8s/ → 跳过模块 6
8. logstash/pipeline/ → 跳过模块 7
```

---

# 模块 1：Pre-commit 脚本

## Bash 版本（.git/hooks/pre-commit）

```bash
#!/bin/sh
echo "============ [工程护栏] 本地代码静态合规性扫描 ============"

# 编译检查（不跑测试）
mvn compile -q || { echo "❌ 编译失败！"; exit 1; }

# Jakarta 命名空间检测（Boot 3.x）
if grep -q "spring-boot-starter-parent" pom.xml 2>/dev/null; then
    BOOT_VERSION=$(grep -A1 "spring-boot-starter-parent" pom.xml | grep "<version>" | sed 's/.*<version>\(.*\)<\/version>.*/\1/')
    case "$BOOT_VERSION" in
        3.*)
            if git diff --cached --name-only | xargs grep -l "import javax\.servlet\|import javax\.annotation\|import javax\.persistence" 2>/dev/null; then
                echo "❌ [CRITICAL] Spring Boot 3.x 检测到 javax.*！替换为 jakarta.*。"
                exit 1
            fi ;;
        2.*)
            if git diff --cached --name-only | xargs grep -l "import jakarta\." 2>/dev/null; then
                echo "❌ [CRITICAL] Boot 2.x 检测到 jakarta.*！检查是否误引入 Boot 3 依赖。"
                exit 1
            fi ;;
    esac
fi

# CONTEXT.md 检查
[ ! -f "CONTEXT.md" ] && echo "❌ 缺少 CONTEXT.md！运行 /plan-java。" && exit 1

# 高危依赖扫描
if git diff --cached --name-only | xargs grep -lE "jedis[^a-z]" 2>/dev/null; then
    echo "⚠️  检测到 Jedis。建议改用 Redisson 或 Lettuce。"
fi

echo "✅ 静态护栏通过。全量测试将在 push 后由 CI 执行。"
exit 0
```

## PowerShell 版本（.git/hooks/pre-commit.ps1）

```powershell
Write-Host "============ [工程护栏] 本地代码静态合规性扫描 ============"

$bootVersion = Select-String -Path "pom.xml" -Pattern "<version>(.*)</version>" | Select-Object -First 1
if ($bootVersion -match "3\..*") {
    $files = git diff --cached --name-only
    foreach ($f in $files) {
        $c = Get-Content $f -Raw -ErrorAction SilentlyContinue
        if ($c -match "import javax\.(servlet|annotation|persistence)") {
            Write-Host "❌ [CRITICAL] Boot 3.x 检测到 javax.*！替换为 jakarta.*。"
            exit 1
        }
    }
}

if (-not (Test-Path "CONTEXT.md")) { Write-Host "❌ 缺少 CONTEXT.md！"; exit 1 }

Write-Host "✅ 静态护栏通过。"
exit 0
```

## 动静分离总结

| 检查 | 本地 pre-commit | CI |
|---|---|---|
| 编译 | `mvn compile -q`（秒） | ✅ |
| Jakarta | grep（毫秒） | ✅ |
| CONTEXT.md | 文件检测（毫秒） | ✅ |
| 全量测试 | ❌ | ✅ |
| OWASP | ❌ | ✅ |
| Trivy | ❌ | ✅ |

---

# 模块 2：安全扫描

## OWASP（pom.xml）

```xml
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <version>10.0.4</version>
    <configuration>
        <failBuildOnCVSS>7</failBuildOnCVSS>
        <formats><format>HTML</format><format>JSON</format></formats>
    </configuration>
    <executions><execution><goals><goal>check</goal></goals></execution></executions>
</plugin>
```

## Trivy（CI step）

```yaml
- name: Build Docker image
  run: docker build -t app:${{ github.sha }} .
- name: Scan with Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: app:${{ github.sha }}
    exit-code: '1'
    severity: 'CRITICAL,HIGH'
```

---

# 模块 3：Springdoc OpenAPI

Boot 3.x：`springdoc-openapi-starter-webmvc-ui:2.6.0`
Boot 2.x：`springdoc-openapi-ui:1.8.0`

```yaml
springdoc:
  api-docs.path: /v3/api-docs
  swagger-ui.path: /swagger-ui.html
```

---

# 模块 4：Docker

## Dockerfile（多阶段）

```dockerfile
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml . && RUN mvn dependency:go-offline -B
COPY src ./src && RUN mvn package -DskipTests -B

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
ENV JAVA_OPTS="-Xms256m -Xmx512m -XX:+UseG1GC"
EXPOSE 8080
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

版本匹配：17→temurin:17、21→temurin:21、11→temurin:11-jre、8→openjdk:8-jre-alpine。

## docker-compose-env.yml

仅生成项目实际用到的中间件（PG/Redis/RocketMQ/Nacos/ELK），每个带 healthcheck 和资源限制。完整模板参考银行转账系统演练中的 `docker-compose-env.yml` 范例。

---

# 模块 5：GitHub Actions CI

```yaml
name: CI
on:
  push:
    branches: [main, master]
  pull_request:
    branches: [main, master]
jobs:
  build-and-test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        java-version: [{检测到的版本}]
    services:
      postgres:
        image: postgres:15-alpine
        env: { POSTGRES_PASSWORD: test, POSTGRES_DB: testdb }
        ports: ["5432:5432"]
        options: --health-cmd pg_isready --health-interval 10s
      redis:
        image: redis:7-alpine
        ports: ["6379:6379"]
        options: --health-cmd "redis-cli ping" --health-interval 10s
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '{vers}', distribution: 'temurin', cache: 'maven' }
      - run: mvn compile
      - run: mvn test
      - run: mvn dependency-check:check -DskipTests
        continue-on-error: true
      - run: docker build -t app:${{ github.sha }} .
      - uses: aquasecurity/trivy-action@master
        with: { image-ref: 'app:${{ github.sha }}', exit-code: '1', severity: 'CRITICAL,HIGH' }
```

注意：Nacos/RocketMQ 不加 CI service，单元测通过 Mock 覆盖。

---

# 模块 6：K8s 清单

Deployment（含 Actuator 探活 + 资源限制）+ Service + Nacos/RocketMQ StatefulSet（含 PVC）。完整模板见银行转账演练中的 K8s 范例。

---

# 模块 7：ELK 配置

- Logstash pipeline：`logstash/pipeline/transfer-log.conf`
- ES 索引模板：`logstash/config/transfer-template.json`
- Spring Boot logback-spring.xml（JSON 格式，含 transactionId/traceId）
- Filebeat K8s DaemonSet + RBAC
- Kibana 运维查询手册：`docs/elk-queries.md`

---

# 反模式（DevOps/安全）

| 反模式 | 纠正 |
|---|---|
| pre-commit 跑全量测试 | 本地仅静态扫描 |
| 硬编码密钥 | K8s Secrets / 环境变量 |
| 镜像 `:latest` | git sha 或语义版本 |
| 无健康检查 | 必须 Actuator liveness/readiness |
| 无资源限制 | 必须 requests/limits |

---

# 回退策略

| 场景 | 退化 |
|---|---|
| `mvn` 不可用 | 尝试 `./gradlew`，都不可用则跳过编译检查 |
| Docker 不可用 | 跳过容器化模块 |
| 已有同名 hook | 备份 `.bak`，提示用户合并 |
| 非 Spring Boot | 跳过 Springdoc |
| Windows | 额外生成 `.ps1` |
