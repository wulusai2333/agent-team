# tdd-java 参考模板与检查表

按需读取。

---

# 交互约束

| 步骤 | 模板 |
|---|---|
| 红灯-1 | `🔴 红灯-1：编译失败 — [方法] 尚不存在，补骨架...` |
| 红灯-2 | `🔴 红灯-2：断言失败 — 骨架就位，开始实现...` |
| 绿灯 | `🟢 绿灯：[测试类] 全部通过 (N/N)` |
| 微重构 | `🔧 微重构：[改动]，测试保持绿色` |
| 深模块检查 | `🏗️ 设计检查：[类名] 接口深度评估 — {通过/拒绝/合并建议}` |
| 提交 | `📦 已提交：feat(模块): [描述]` |
| Bug-复现 | `🐛 阶段 A：已复现 Bug — [测试名] → [实际结果]` |
| Bug-缩小 | `🔍 阶段 B：问题已缩小至 [模块.方法:行号]` |
| Bug-假设 | `💡 阶段 C：[N] 个假设：[列表]。验证中...` |
| Bug-修复 | `✅ 阶段 E：修复确认，回归测试通过` |

---

---

# codebase-design：深模块设计检查

在每个 TDD 循环中创建新类或新接口之前，强制执行此检查。拒绝"为了拆分而拆分"的浅模块。

## 概念

| 概念 | 定义 | Java 对应 |
|---|---|---|
| 深模块 | 接口极小，内部极复杂——一个方法背后藏着大量逻辑 | 一个好的 `@Service`：`void transfer(TransferRequest)`，内部做了幂等+锁+扣款+审计 |
| 浅模块 | 接口复杂，内部极简单——一堆方法全是 getter/setter 或转发 | 一个 `XxxHelper` 类有 12 个方法，每个方法只有 2 行，全部转发给别的类 |

## 深度检查（创建新类前）

```
这个新类：
  1. 公开方法有几个？→ ≥5 个 → 可能太复杂，考虑拆
  2. 是否大部分方法是转发/委托/简单 getter？→ 是 → 浅模块，拒绝创建，合并回调用方
  3. 去掉这个类，逻辑内联回调用方，是否更清晰？→ 是 → 不创建
  4. 这个类的接口是否隐藏了足够的内部复杂度？→ 是 → 深模块，允许创建
```

## 反模式

| 反模式 | 例子 | 纠正 |
|---|---|---|
| Manager/Helper/Util 泛滥 | `TransferManager`, `TransferHelper`, `TransferUtil` 同时存在 | 合并成一个 `TransferService` |
| 一步一接口 | `IAccountRepo` → `AccountRepoImpl` 只有 1:1 实现 | 只有多个实现时才需要接口 |
| 微服务崇拜 | 一个简单的查询功能拆成 3 个微服务 | 用模块拆分，不用服务边界拆分 |
| Spring Bean 碎片化 | 5 个 `@Service` 每个只有 1 个方法，互相注入 | 合并，减少 Bean 图复杂度 |

## 深度优先 dep 分类（参考 DEEPENING.md）

| 依赖类型 | 定义 | 测试策略 |
|---|---|---|
| 进程内（in-process） | 纯 Java 逻辑，不涉及 IO | 纯单元测试，毫秒级 |
| 本地可替换（local-substitutable） | 涉及 IO 但可替换（H2 代替 PG） | 切片测试 |
| 远端自有（remote-owned） | Feign 调自己的微服务 | `@MockBean` |
| 外部系统（external） | 第三方 API、支付网关 | Mockito/WireMock |

**重构策略**：把尽可能多的代码移到进程内层，减少远端依赖的代码量。

---

---

# 断点续传规则

```bash
git branch --show-current
git status --short
git log -1 --oneline                # 提取 Issue 号
find src/test -name "*Test.java" | head -10
```

**跨平台**：`find` → Windows 用 `Get-ChildItem -Recurse -Filter "*Test.java"`。

---

# 多模块检测

```bash
grep -c '<module>' pom.xml          # >0 则多模块
find . -name "XxxTest.java" -path "*/test/*"   # 定位模块
mvn test -pl :order-service -Dtest=AccountServiceTest   # 指定模块
```

单模块直接用 `mvn test -Dtest=XxxTest`。

---

# 测试分层详情

```
纯单元（默认）：@ExtendWith(MockitoExtension.class)，不启 Spring，毫秒
切片（按需）：@WebMvcTest / @DataJpaTest / @JsonTest，秒级
集成（谨慎）：@SpringBootTest，十秒级，仅关键路径
```

选择原则：能用纯单元不切片，能切片不全量。

---

# Mock 护栏 + 数据隔离

## 必须 Mock

| 组件 | 方案 |
|---|---|
| Feign | `@MockBean` |
| Nacos/Consul | `spring.cloud.nacos.discovery.enabled=false` |
| 配置中心 | `application-test.yml` 本地配置 |
| MQ | `@MockBean` Template/Listener |
| Redis | 单元测 Mock；集成测 embedded-redis（Lua 才用 Testcontainers） |
| DB | 单元测 Mock；集成测 H2（JSONB/全文检索才用 Testcontainers） |
| 外部 API | `@MockBean` 或 WireMock |

## Testcontainers 前置

```bash
docker info > /dev/null 2>&1 && echo "可用" || echo "不可用"
```

不可用 → H2/内嵌。国内镜像拉取失败 → 预配镜像加速。只对 JSONB/Lua 等特定功能启用。

## 数据隔离

```java
@SpringBootTest @Transactional @Rollback   // 首选
@DataJpaTest                                // Repository 层
```

禁止：连物理 DB、TRUNCATE、多人共用实例。

## Mock 速查

```java
@MockBean private UserFeignClient feign;     // ✅ 系统边界
@Mock private UserRepository repo;           // ✅ 数据层
@InjectMocks private UserService svc;        // ✅ 被测对象
@MockBean private OrderService svc;          // ❌ 内部类不 Mock
```

---

# 红灯双阶段

| 阶段 | 形态 | 行动 |
|---|---|---|
| 红灯-1 | `cannot find symbol` | 写骨架：`return null;` 或 `throw new UnsupportedOperationException()` |
| 红灯-2 | `Expected: X, Actual: null` | 写最小实现 |

---

# diagnosing-bugs：6 阶段诊断循环

当测试红灯不是"方法不存在"（那是正常 TDD 的红灯-1）而是**已有代码的预期行为被打破**时，自动切入此模式。

## 阶段 A：复现

- 收到 Bug 报告后的第一件事：写一个**精确复现 Bug 的失败测试**
- 测试名用 `should_Reproduce_Bug_{Issue号}`，加 `@DisplayName` 描述
- 如果无法复现 → 报告用户"当前代码无法复现此 Bug"，停止诊断

## 阶段 B：缩小

二分法定位故障点：

```
1. 整条链路是否全部挂？→ 是 → 检查公共依赖（连接池/配置中心/注册中心）
2. 单个模块挂？→ 对比该模块的最近提交历史
3. 单个方法挂？→ 检查该方法最近的修改
4. 某行代码挂？→ 检查该行的前置条件是否被破坏
```

Spring Boot 缩小策略：
- Controller 层 → `@WebMvcTest` 隔离测试
- Service 层 → `@ExtendWith(MockitoExtension.class)` 纯单元测试
- Repository 层 → `@DataJpaTest` 隔离测试
- 跨服务调用 → `@MockBean` 逐个 Mock 剔除

## 阶段 C：假设

提出**不超过 3 个**可验证的根因假设。每个假设必须有对应的验证方法：

```
假设 1：[原因] → 验证：[写什么样的测试/日志来证明或推翻]
假设 2：[原因] → 验证：[...]
假设 3：[原因] → 验证：[...]
```

常见的 Java/Spring 根因类别：
- 事务边界错误（`@Transactional` 没生效、自调用失效）
- 线程安全问题（`@Service` 中可变状态、缓存并发）
- 依赖版本冲突（依赖树上同一 JAR 有 2 个版本）
- 配置污染（`application.yml` 被 Profile 覆盖）
- Nacos 配置推送延迟 / 服务列表未刷新

## 阶段 D：插桩

添加临时诊断代码来验证假设。**插桩代码是不提交的**——用 `// FIXME: DIAGNOSTIC - remove after debugging` 标记：

```java
// 临时日志插桩
log.warn("DIAGNOSTIC: transactionId={}, balance_before={}, lock_owner={}",
    txId, before, lockOwner);

// 临时 AOP 插桩（如需要）
@Around("execution(* com.bank.transfer..*(..))")
public Object profile(ProceedingJoinPoint pjp) throws Throwable {
    long start = System.nanoTime();
    Object result = pjp.proceed();
    log.warn("DIAGNOSTIC: {} took {}ms", pjp.getSignature(), (System.nanoTime()-start)/1_000_000);
    return result;
}
```

## 阶段 E：修复 + 回归测试

- 确认根因后，写最小修复代码
- **保留复现 Bug 的那个测试**作为回归测试（改名 `should_{描述}_When_{条件}`）
- 运行全量测试确认没有引入新问题

## 阶段 F：清理 + 回顾

- 删除所有 `DIAGNOSTIC` 标记的插桩代码
- 问自己（Agent）："这个 Bug 应该在哪个更早的阶段被发现？"
  - 如果 TDD 阶段能发现 → 补一个测试用例
  - 如果 code review 能发现 → 在反模式清单里加一条
  - 如果只有生产环境能发现 → 在 guardrails 里加一个检查
- 提交修复，commit message 格式：`fix(模块): [Issue#N] {Bug简述}`

---

## 步骤 2.1：写测试

```java
@ExtendWith(MockitoExtension.class)
class AccountServiceTest {
    @Mock private AccountRepository repo;
    @InjectMocks private AccountServiceImpl svc;

    @Test @DisplayName("getBalance：用户存在 → 返回余额")
    void should_ReturnBalance_When_UserExists() {
        when(repo.findBalanceById(1L)).thenReturn(BigDecimal.valueOf(100.0));
        assertEquals(BigDecimal.valueOf(100.0), svc.getBalance(1L));
    }
}
```

## 步骤 2.2-2.3：红灯

```bash
mvn test -pl :module -Dtest=AccountServiceTest   # 红灯-1: cannot find symbol
# → 补骨架：public BigDecimal getBalance(Long id) { return null; }
# → 红灯-2: Expected 100.0, Actual null
```

## 步骤 3：绿灯

```java
public BigDecimal getBalance(Long userId) {
    return repo.findBalanceById(userId);
}
```

## 步骤 4：微重构

只改刚写的代码（硬编码→常量、`new`→`@Autowired`）。每改一步重跑测试。宏观重构留给 `/code-review-java`。

## 步骤 5：提交

```bash
git add -A && git commit -m "feat(模块): [Issue#N] 简短描述"
```

---

# Spring Boot 集成测试模板

```java
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired private MockMvc mvc;
    @MockBean private UserService svc;

    @Test @DisplayName("POST /users → 201 + JSON")
    void should_Return201_When_ValidRequest() throws Exception {
        when(svc.createUser(any())).thenReturn(new User(1L, "张三", "x@x.com"));
        mvc.perform(post("/users").contentType(APPLICATION_JSON)
            .content(new ObjectMapper().writeValueAsString(req)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.name").value("张三"));
    }
}
```

---

# 构建命令速查

| 场景 | Maven（单模块） | Maven（多模块） | Gradle |
|---|---|---|---|
| 单测 | `mvn test -Dtest=X` | `mvn test -pl :m -Dtest=X` | `./gradlew test --tests X` |
| 全量 | `mvn test` | `mvn test`（CI） | `./gradlew test` |
| 打包 | `mvn package -DskipTests` | 同 | `./gradlew build -x test` |

---

# 反模式

| 反模式 | 纠正 |
|---|---|
| `verify(repo).save()` 断言调用次数 | 验证行为结果 |
| 写 Issue 没要求的测试 | 只覆盖当前 AC |
| 先写所有测试再写所有实现 | 一个接缝 → 一个测试 → 一个实现 |
| Mock 了自己的 Service | 只 Mock 系统边界 |
| 每个测试 `@SpringBootTest` | 优先 `@ExtendWith(MockitoExtension.class)` |
