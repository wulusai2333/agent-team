# code-review-java 参考检查表与子 Agent 提示词

按需读取。

---

# 断点续传

```bash
git log -1 --oneline    # 确认基准点
# 检查是否有子Agent输出文件 → 有则直接聚合，无则重新发起
```

---

# Java/Spring 专项基线

## 命名约定

| 检查项 | 正确 | 错误 |
|---|---|---|
| 类名 | `PascalCase` | `transferService` |
| 方法名 | `camelCase` | `FindById()` |
| 常量 | `UPPER_SNAKE` | `maxRetry` |
| 包名 | 全小写 | `com.bank.Transfer` |

## Spring 分层反模式

- Controller 含业务逻辑 → 逻辑下沉 Service
- Service 跨层调 Repository → 通过接口调用
- 循环依赖（A→B→A）→ 提取公共层或用事件解耦
- Field Injection（`@Autowired private X`）→ 用构造器注入
- `new` 创建 Bean → 让 Spring 管理
- `@Transactional` 加在 Controller → 只加 Service
- `findAll()` 无分页 → 用 `Pageable`

## Spring Cloud 反模式

- Feign 无熔断 → 必须配 Sentinel fallback
- 硬编码服务地址 → 通过 Nacos + Feign
- 配置散落代码 → 收口到 Nacos Config
- MQ 消费无幂等 → 必须幂等（金融系统红线）

## 测试反模式

- 全量 `@SpringBootTest` → 优先 Mockito
- 测试依赖执行顺序 → 每个独立 `@BeforeEach`
- Mock 内部类 → 只 Mock 边界
- 无 Given-When-Then 结构

## 金融加检（如适用）

- 金额必须 `BigDecimal`，禁止 `float/double`
- 写操作必须幂等
- 状态变更必须审计日志（`created_at`/`updated_at`）
- 禁止 `System.out.println`，必须 SLF4J + `transactionId`

---

# Fowler 坏味道基线

| 坏味道 | 判断 | 修复 |
|---|---|---|
| 神秘命名 | 变量名不揭示含义 | 重命名 |
| 重复代码 | 相同逻辑出现多次 | 提取公共方法 |
| 依恋情节 | 方法频繁访问另一个对象 | 移动方法 |
| 数据泥团 | 相同字段总是结伴 | 提取类型 |
| 基本类型偏执 | String 代替领域概念 | 值对象 |
| 重复 Switch | 同一判断出现多次 | 多态 |
| 霰弹式修改 | 一个变化散落多处 | 收拢 |
| 发散式变化 | 一个类多种原因修改 | 拆分 |
| 夸夸其谈通用性 | 为"未来"加的抽象 | 删除 |
| 消息链 | `a.b().c().d()` | 委托方法 |
| 中间人 | 类只做转发 | 去掉中间层 |
| 拒绝遗赠 | 子类忽略继承 | 改用组合 |

规则：仓库规范 > 基线。工具已强制的（Checkstyle）不重复报告。

---

# 子 Agent 并行提示词

## Standards 子 Agent

```
基于 git diff <fixed-point>...HEAD，报告每个文件/hunk：
(a) 违反编码标准处——引用标准文件+规则名
(b) Java/Spring 反模式或坏味道——命名并引用代码片段
(c) 区分硬违规和判断性违规
跳过工具已强制的检查。400 词以内。
```

## Spec 子 Agent

```
基于 git diff <fixed-point>...HEAD 和 Spec 文件：
(a) Spec 要求但缺失/不完整的需求——引用 Spec 原文
(b) diff 中有但 Spec 没要求的行为（范围蔓延）
(c) 看似实现但看起来错误的——引用 Spec 原文
400 词以内。无 Spec 源则跳过并报告。
```
