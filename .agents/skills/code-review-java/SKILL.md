---
name: code-review-java
description: Java/Spring Boot 双轴代码评审。Standards（编码规范 + Spring 分层 + Fowler 坏味道）与 Spec（对照 Issue/PRD 验收标准）。两轴并行，分别报告。
---

# Java/Spring Boot 双轴代码评审

基于原版 `/code-review` 的双轴模型，增加 Java/Spring 生态特化检查。

---

# 断点续传规则

被中断后重新唤醒时：

```
1. git log -1 --oneline → 确认上次评审的基准点
2. 如果子 Agent 结果已写入临时文件 → 读取并直接进入聚合阶段
3. 如果子 Agent 未完成 → 重新发起并行评审
```

> "恢复会话。上次评审基准点：{commit}。Standards 子Agent {已完成/未完成}，Spec 子Agent {已完成/未完成}。继续聚合报告。"

---

# 交互约束

| 步骤 | 汇报模板 |
|---|---|
| 固定点确认 | `评审基准点：{commit/branch}。Diff 包含 {N} 个文件，{M} 次提交。` |
| Spec 源定位 | `Spec 来源：{Issue/PRD 路径}。`（或：`未找到 Spec 源，Spec 轴跳过。`） |
| 子 Agent 并行启动 | `Standards 和 Spec 子评审并行启动中...` |
| 聚合完成 | `双轴评审完成。Standards 发现 {N} 项，Spec 发现 {M} 项。` |

---

# 完成后交接

评审报告呈现后，明确指引用户下一步：

> 评审完成。下一步：
> 1. 根据 Standards 报告修复编码规范问题
> 2. 根据 Spec 报告补全遗漏的验收标准
> 3. 修复后重新运行 `/code-review-java` 确认
> 4. 全部通过后推送并创建 PR

---

## 双轴模型

- **Standards** — 代码是否符合本项目的编码规范 + Spring 分层约定 + Fowler 坏味道基线？
- **Spec** — 代码是否忠实实现了 Issue/PRD 的验收标准？

两轴作为并行子 Agent 运行，上下文互不污染。

---

## Process

### 1. 确定固定点

用户指定的基准点 — commit SHA、分支名、tag、`main`、`HEAD~5` 等。

```bash
git diff <fixed-point>...HEAD
git log <fixed-point>..HEAD --oneline
```

确认固定点可解析且 diff 非空后继续。

### 2. 识别 Spec 来源

按优先级：
1. Commit 消息中的 Issue 引用（`#123`、`Closes #45`）
2. 用户传入的路径
3. `docs/prd/` 下匹配的 PRD 文件
4. 都没有 → Spec 轴跳过，报告"无可用的规格说明"

### 3. 识别 Standards 来源

- `CLAUDE.md` 中的编码约定
- `CONTRIBUTING.md` 或 `CODING_STANDARDS.md`
- `checkstyle.xml` 或 `spotless` 配置
- 项目 `CONTEXT.md` 中的领域术语

此外，Standards 轴**始终携带以下 Java/Spring 专项基线**。

---

## Java/Spring 专项基线

### 命名约定（对照项目标准）

> 以下为 Java 社区约定，如项目有自己的规范文件则以项目为准。

| 检查项 | 正确 | 错误 |
|---|---|---|
| 类名 | `PascalCase`：`TransferService` | `transferService`、`transfer_service` |
| 方法名 | `camelCase`：`findByTransactionId()` | `FindByTransactionId()`、`find_by_txn()` |
| 常量 | `UPPER_SNAKE`：`MAX_RETRY_COUNT` | `maxRetryCount` |
| 包名 | 全小写：`com.bank.transfer` | `com.bank.Transfer` |
| 测试类名 | `{被测类}Test`：`TransferServiceTest` | `TestTransferService` |
| 测试方法 | `should_{行为}_When_{条件}` 或 `{方法}_{场景}_{预期}` | `test1`、`testTransfer` |

### Spring 分层约定

| 反模式 | 表现 | 纠正 |
|---|---|---|
| **Controller 包含业务逻辑** | Controller 方法里直接操作 Repository、拼装数据 | 逻辑下沉到 Service，Controller 只做参数校验 + 调用 + 返回 |
| **Service 跨层调用** | Service 直接调另一个 Service 的 Repository | 通过 Service 接口调用，不穿透 |
| **Repository 裸用 EntityManager** | 手写 SQL 散落在 Service 里 | 收口到 Repository 接口或 `@Query` 注解 |
| **循环依赖** | AService 注入 BService，BService 注入 AService | 提取公共逻辑到第三个 Service，或用事件解耦 |
| **Field Injection** | `@Autowired private XxxService service;` | 改用构造器注入：`private final XxxService service;` + `@RequiredArgsConstructor` |
| **new 创建 Spring Bean** | `new XxxService()` 在 Controller/Service 里 | 让 Spring 管理生命周期，通过 `@Autowired` 注入 |
| **事务边界错误** | `@Transactional` 加在 Controller 上 | `@Transactional` 只加在 Service 方法上 |
| **无分页查询** | `findAll()` 不加 `Pageable` | 列表查询一律用 `Pageable`，防止 OOM |

### Spring Cloud 微服务专项

| 反模式 | 表现 | 纠正 |
|---|---|---|
| **Feign 调用无熔断** | `@FeignClient` 没有 fallback | 必须配置 Sentinel/Hystrix fallback |
| **硬编码服务地址** | 写死 `http://10.0.0.1:8080` | 通过 Nacos 服务发现 + Feign 接口调用 |
| **配置散落在代码里** | `application.yml` 之外在代码里 hardcode 配置值 | 所有配置收口到 Nacos Config 或 `application.yml` |
| **RocketMQ 消费无幂等** | Consumer 不检查消息是否已处理 | 金融系统必须做幂等（参考 ADR-0002） |

### 测试专项

| 反模式 | 表现 | 纠正 |
|---|---|---|
| **全量 `@SpringBootTest`** | 每个测试都启动整个 Spring 容器 | 优先 `@ExtendWith(MockitoExtension.class)`，集成测才用 `@SpringBootTest` |
| **测试依赖执行顺序** | 测试 B 依赖测试 A 写入的数据 | 每个测试独立准备数据（`@BeforeEach`），不依赖执行顺序 |
| **Mock 内部类** | Mock 了自己写的 `OrderService` | 只 Mock 系统边界（Feign/MQ/外部API），不 Mock 内部类 |
| **无 Given-When-Then** | 测试方法里看不出准备-执行-断言的结构 | 用空行分隔三段，或用 BDDMockito 风格 |

### 金融系统加检（如项目是金融领域）

| 检查项 | 红线 |
|---|---|
| **金额类型** | 金额必须用 `BigDecimal`，禁止 `float`/`double` |
| **幂等** | 所有写操作必须有幂等机制（唯一索引/Redis SETNX/乐观锁），参考 ADR-0002 |
| **审计** | 关键状态变更必须记录 `created_at`、`updated_at`、操作来源 |
| **日志** | 禁止 `System.out.println`，必须用 SLF4J。金融交易日志必须包含 `transactionId` |

### Fowler 坏味道基线（通用，始终检查）

每种坏味道对照 diff 检查：

- **神秘命名** — 变量/方法名没有揭示其含义 → 重命名
- **重复代码** — 同一逻辑在多个 hunk 中出现 → 提取公共方法
- **依恋情节** — 方法频繁访问另一个对象的数据 → 移动到数据所在类
- **数据泥团** — 同样几个字段总是结伴出现 → 提取为独立类型
- **基本类型偏执** — 用 String/Integer 代替领域概念 → 创建值对象
- **重复 Switch** — 同一类型判断出现在多处 → 多态替代
- **霰弹式修改** — 一个变化导致散落在多处的修改 → 收拢到一处
- **发散式变化** — 一个类被多种不相关的原因修改 → 拆分类
- **夸夸其谈的通用性** — 为未来需求加的抽象 → 删除，等真正需要时再加
- **消息链** — 长链式调用 `a.b().c().d()` → 改为委托方法
- **中间人** — 类主要只做了转发 → 去掉中间层
- **拒绝的遗赠** — 子类忽略大部分继承 → 改用组合

**规则**：仓库规范优先于基线。工具已强制的检查（Checkstyle/SpotBugs）不重复报告。

---

## 子 Agent 并行调度

发送两个并行 Agent：

### Standards 子 Agent

提示词包含：
- 完整 diff 命令和 commit 列表
- 找到的 Standards 源文件
- **完整的 Java/Spring 专项基线**（本文档全部内容）
- **Fowler 坏味道基线**

简报：
> "报告每个文件/hunk——(a) 违反编码标准的地方：引用标准文件 + 规则名；(b) 任何 Java/Spring 反模式或坏味道：命名并引用代码片段；(c) 区分硬违规和判断性违规。跳过工具已强制的检查。400 词以内。"

### Spec 子 Agent

提示词包含：
- diff 命令和 commit 列表
- Spec 文件路径或已获取的内容

简报：
> "报告——(a) Spec 要求但缺失或不完整的需求；(b) diff 中有但 Spec 没要求的行为（范围蔓延）；(c) 看似实现了但实现看起来错误的需求。逐条引用 Spec 原文。400 词以内。"

---

## 聚合报告

两个子 Agent 结果并列展示：

```
## Standards
{Standards 子 Agent 报告}

## Spec
{Spec 子 Agent 报告}

---

总结：Standards 发现 N 项，最严重为 [xxx]。Spec 发现 M 项，最严重为 [xxx]。
```

不合并、不重排序。两轴独立，防止一轴掩盖另一轴。
