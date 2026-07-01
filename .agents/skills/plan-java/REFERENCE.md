# plan-java 参考模板与检查表

此文件是 `/plan-java` 的模板仓库。Agent 在执行到具体步骤时按需读取。

---

# 步骤 0：需求清晰度判定

## decision-mapping：当用户完全不知道从哪开始

当用户的需求只有一句话甚至几个词（如"我想做一个系统"），不要强行进入三维盘问。此时用户需要的不是 PRD，而是**搞清楚自己到底要什么**。

**流程**：

1. 把模糊需求拆成 4-6 个**探索路径**，每条路径是一个可独立推进的调查任务：

```
探索路径 1：竞品调研 — 同类系统怎么做？有哪些成熟方案？
探索路径 2：用户场景 — 谁会用它？一天中用几次？关键操作是什么？
探索路径 3：技术可行性 — 核心技术难点在哪？有没有已知的坑？
探索路径 4：最小可行版本 — 第一版只做一件什么事才算"能用"？
探索路径 5（可选）：原型验证 — 哪个逻辑最不确定？写个终端原型跑一下
探索路径 6（可选）：决策会议 — 汇总前 5 条，做出关键选型决定
```

2. 每条路径的输出是一个极短的总结（≤200 词），不做详细设计
3. 所有路径走完后，汇总为一个"决策摘要"，此时用户的需求已经清晰，进入正常的三维盘问

## prototype：当状态模型或业务逻辑存疑时

当用户知道要做什么但不确定怎么做（"扣款逻辑应该是先锁再扣还是先查再冻？"），不写 PRD。写一个**丢弃式原型**。

**Java 原型模板**：

```java
// PROTOYPE.java — 验证后删除，不进入正式代码
import java.math.BigDecimal;
import java.util.*;

public class TransferPrototype {
    // 纯逻辑：不连数据库、不连 Redis、不启动 Spring
    static class Account {
        String id;
        BigDecimal balance;
        BigDecimal frozen;  // 冻结金额
    }

    static enum TransferResult { SUCCESS, INSUFFICIENT, DUPLICATE }

    // 被验证的核心逻辑：先冻结 → 扣款 → 解冻
    static TransferResult transfer(Account from, Account to, BigDecimal amount) {
        if (from.balance.compareTo(amount) < 0) return TransferResult.INSUFFICIENT;
        from.balance = from.balance.subtract(amount);
        from.frozen = from.frozen.add(amount);  // 冻结
        to.balance = to.balance.add(amount);
        from.frozen = from.frozen.subtract(amount);  // 解冻
        return TransferResult.SUCCESS;
    }

    // 终端交互：用户手动输入，观察状态变化
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        Account a1 = new Account(); a1.id = "A1"; a1.balance = new BigDecimal("1000");
        Account a2 = new Account(); a2.id = "A2"; a2.balance = new BigDecimal("500");
        System.out.println("A1: " + a1.balance + " | A2: " + a2.balance);
        System.out.print("转账金额: ");
        BigDecimal amt = new BigDecimal(sc.nextLine());
        TransferResult r = transfer(a1, a2, amt);
        System.out.println("结果: " + r);
        System.out.println("A1: " + a1.balance + " (冻结:" + a1.frozen + ") | A2: " + a2.balance);
    }
}
```

**运行验证**：

```bash
javac TransferPrototype.java && java TransferPrototype
```

**规则**：
- 原型用纯 Java SE，不引入任何框架依赖
- 用 `Scanner` + `System.out` 做终端交互
- 用 `BigDecimal`、`record`、`enum` 建模领域逻辑
- 验证通过后**立即删除**原型文件，不提交到 Git
- 将验证结论写入 PRD 的"实现决策"中，标注"经原型验证"

---

---

# 断点续传规则

```
1. CONTEXT.md 是否存在？→ 已存在则跳过步骤 3
2. docs/adr/ 下最大编号？→ 新 ADR 从 N+1 开始
3. gh issue list --label ready-for-agent → 列出未完成的 Issue
4. 是否有 PRD 草稿？→ 读取并接续
```

**跨平台命令**：

| Linux/macOS | Windows PowerShell |
|---|---|
| `grep -c '<module>' pom.xml` | `Select-String -Pattern '<module>' pom.xml` |
| `ls docs/adr/*.md` | `Get-ChildItem docs/adr/*.md` |

---

# 步骤 1：三维一体盘问

**规则：每轮最多 3 问，一问一答。**

**维度 1 — 业务流与异常分支**：主路径？下游超时如何降级？Redis 挂了如何容灾？

**维度 2 — 技术基座**：Java 版本？Boot/Cloud 版本？涉及哪些现有模块？

**维度 3 — 性能与一致性**：分布式事务（Seata AT / MQ 事务消息）？高并发扣减（Lua / 行级锁）？缓存模式（Cache-Aside？删缓存还是异步淘汰？）

---

# 步骤 2：版本校验（变化触发）

**先读 CONTEXT.md 基线。** 如果基线存在且用户没有提"升级/引入/新中间件/换版本/迁移"，复用基线跳过全量审计。否则激活以下检查。

## 2.1 Jakarta 合规

| Java | Boot | 命名空间 |
|---|---|---|
| 8/11 | 2.x | `javax.*`（无风险）|
| 17/21 | 3.x | `jakarta.*`（禁止 `javax.*`）|

Boot 3.x 禁止：Swagger 2.x / Knife4j 旧版 / 旧版 sharding-jdbc。替代：SpringDoc / Knife4j 4.x / ShardingSphere 5.x。

## 2.2 三位一体对齐

| Boot | Cloud | Cloud Alibaba |
|---|---|---|
| 2.7.x | 2021.0.x | 2021.0.x |
| 3.0.x | 2022.0.x | 2022.0.x |
| 3.2.x | 2023.0.x | 2023.0.x |

## 2.3 中间件检查

- MySQL → Boot 3.x 用 `mysql-connector-j`，不是 `mysql-connector-java`
- PostgreSQL → 驱动 42.x，无已知冲突
- Redis → 默认 Lettuce。Redisson 检查版本兼容。禁止裸 Jedis
- RocketMQ → Client 版本对齐 Stream Binder
- Nacos → Server ≥ Client
- Seata → Client/Server 主版本一致
- 配置中心 → Boot 3.x 需 `spring-cloud-starter-bootstrap`

## 2.4 数据一致性默认

- Redis + DB → Cache-Aside（先更新 DB，再删缓存）
- 跨服务 → 默认最终一致性（MQ 事务消息），强一致性才评估 Seata AT

## 2.5 禁止引入

- ❌ Beta/Alpha/Milestone/RC
- ❌ 超过 12 个月未更新的依赖
- ❌ 与 Boot 主版本不兼容的 Starter
- ❌ 裸 Jedis

## 2.6 私服检查

```bash
cat ~/.m2/settings.xml 2>/dev/null | grep -E '<mirror>|<repository>' || echo "未检测到私服"
mvn validate -q 2>&1 | grep -i "could not transfer" && echo "⚠️ 私服不可达"
```

---

# 步骤 3：CONTEXT.md 模板

```markdown
# 项目全局业务与架构上下文

## 领域术语

**{术语名}**：{一句话定义}
_Avoid_: {应避免使用的同义词}

## 技术栈基线

| 组件 | 版本/方案 | 备注 |
|---|---|---|
| JDK | {版本} | |
| Spring Boot | {版本} | |
| Spring Cloud | {版本} | |
| Spring Cloud Alibaba | {版本} | |
| 数据库 | {版本} | {连接池} |
| 缓存 | {版本+客户端} | |
| 消息队列 | {版本} | |
```

只放术语和技术基线。无变动时不触碰。

---

# 步骤 4：ADR 模板

三条件：难以逆转 + 令人困惑 + 真实权衡。不满足则跳过。

```markdown
# {简短标题}

## 背景
{1-3 句话}

## 选型
| 组件 | 版本/方案 | 理由 |
|---|---|---|
| ... | ... | ... |

## 兼容性风险评估
| 风险 | 等级 | 缓解 |
|---|---|---|
| ... | 高/中/低 | ... |

## 备选方案
| 方案 | 为何不选 |
|---|---|
| ... | ... |

## 后果
{运维注意事项、JVM 参数等}
```

---

# 步骤 5：PRD 模板

```markdown
# {功能名称}

## 问题陈述
{用户角度}

## 解决方案
{用户角度}

## 用户故事
1. As a {角色}, I want {功能}, so that {价值}
（涵盖正常路径 + 异常路径）

## 实现决策
- 模块划分、接口契约、数据模型
- 数据一致性策略
- 降级策略
- 版本基线

## 测试策略
- 接缝层级：{单元/切片/集成}
- Mock 策略：{需 Mock 的边界}
- 关键集成路径

## 不在范围内
{明确排除}
```

---

# 步骤 6：Issue 模板 + 发布

```markdown
## 父级
{PRD Issue 引用}

## 要构建什么
{一句话端到端行为}

## 技术缝隙
- 涉及模块：{名称}
- 数据层：{表/字段/缓存 Key}
- 接口层：{API 路径}
- 依赖组件：{MQ/RPC}
- Mock 策略：{需 Mock 的边界}

## 验收标准（Given-When-Then）
- [ ] Given {前置}, When {操作}, Then {预期}

## 阻塞关系
{依赖或"无"}

---
💡 Agent 快速接单指令：`/tdd-java 抓取并执行当前 Issue #[N]`
```

发布：`gh issue create --title "..." --body "..." --label "ready-for-agent"`。按依赖顺序，无阻塞优先。

---

# 反模式警告（架构与规划阶段）

| 反模式 | 表现 | 纠正 |
|---|---|---|
| 过度设计 | 查询接口设计了 CQRS + ES | 最简单架构满足当前需求 |
| 版本跟风 | 选最新版不考虑生态成熟度 | LTS 优先，GA 优先 |
| 忽略降级 | PRD 只写正常路径 | 每个故事对应降级策略 |
| 水平切片 | Issue 按"建表→DAO→Service"拆 | 每个 Issue 贯穿全层 |
| ADR 泛滥 | 每次选型都写 ADR | 只对三条件成立的写 |

---

# 回退策略

| 场景 | 退化行为 |
|---|---|
| `gh` CLI 不可用 | Issue 改为本地 `.scratch/<feature>/issues/` Markdown |
| 私服不可达 | `mvn validate` 后提示用户检查 VPN/settings.xml |
| 用户拒绝盘问 | 基于已有信息继续，标注"未经确认的假设" |
| 基线版本号无法解析 | 不跳过审计，激活全量版本校验 |
