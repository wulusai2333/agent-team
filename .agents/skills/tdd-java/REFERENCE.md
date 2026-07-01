# tdd-java 参考模板与检查表

按需读取，不在技能启动时全部加载。

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

# 核心闭环代码模板

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
