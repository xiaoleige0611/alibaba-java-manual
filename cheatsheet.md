# 速查卡：按手册做决策

## 规约级别怎么用
| 级别 | 含义 | Code Review 时的处理 |
|---|---|---|
| 【强制】 | 违反就可能出故障 | 必须修改，阻断合并 |
| 【推荐】 | 最佳实践 | 原则上修改，偏离时要写明理由 |
| 【参考】 | 经验建议 | 作为讨论依据 |

## 当 X 时，做 Y
| 当… | 做… | 因为… |
|---|---|---|
| 比较两个 Integer/Long | `equals` 或 `Objects.equals` | 超出 -128~127 后 `==` 比较的是引用 |
| 判断浮点数是否相等 | 误差范围或 BigDecimal | 二进制无法精确表示十进制小数 |
| 构造 BigDecimal | `new BigDecimal("0.1")` / `valueOf` | double 构造会丢失精度 |
| 定义 POJO 字段 | 用包装类型、无默认值、布尔字段不加 is | null 能表达“调用失败”；避免框架误判属性名 |
| 需要线程 | 用 ThreadPoolExecutor 加有界队列和线程名 | Executors 是无界的，会 OOM |
| 加锁 | 先 `lock()`，紧跟 try，在 finally 中 `unlock()` | 防止对未持有的锁调用 unlock |
| 并发更新记录 | 冲突率低于 20% 用 version 乐观锁（重试 ≥ 3 次），否则或涉及资金用悲观锁 | 兼顾吞吐与安全 |
| 遍历时删除元素 | 用 Iterator.remove | foreach 中删除会抛 ConcurrentModificationException |
| 集合转数组 | `toArray(new T[0])` | 性能最好、类型安全 |
| 初始化 HashMap | 容量 = n / 0.75 + 1，无法确定时用 16 | 避免多次 resize |
| if-else 超过 3 层 | 改用卫语句、策略模式或状态模式 | 可读、可测 |
| 外部传入的 String 用于 switch | 先判断 null | switch(null) 会抛 NPE |
| 高并发下判断终止条件 | 用区间判断（`<= 0`），不用 `== 0` | 等值判断可能被“击穿” |
| 调用 RPC 或二方包 | catch Throwable | 防止 NoSuchMethodError |
| 设计跨应用接口 | 返回 Result；对外 http 接口用错误码 | 异常在 RPC 边界上不可靠 |
| 写日志 | SLF4J、`{}` 占位符、debug 先判断级别、带现场信息和 e | 性能好，便于排查 |
| 用户输入参数错误 | 记 warn | error 会引发频繁报警 |
| 存金额或小数 | Java 用 BigDecimal，表字段用 decimal | float/double 有精度损失 |
| 需要分页的深度查询 | 用延迟关联 | 否则 offset 越大越慢 |
| SQL 传参 | 用 `#{}` | `${}` 有 SQL 注入风险 |
| 查询参数超过 2 个 | 封装成 Query 对象 | 禁止用 Map 传参 |

## 阈值与默认值
| 项 | 值 |
|---|---|
| 单行字符数 | ≤ 120 |
| 方法行数 | ≤ 80 |
| 缩进 | 4 个空格，不用 tab |
| if-else 嵌套 | ≤ 3 层 |
| 乐观锁 | 冲突率 < 20%，重试 ≥ 3 次 |
| HashMap 负载因子 | 0.75，默认容量 16 |
| 单测覆盖率 | 语句 70%；核心模块的语句和分支都是 100% |
| 日志保留 | ≥ 15 天；敏感操作日志 ≥ 6 个月 |
| varchar 长度 | ≤ 5000，超过就用 text 并拆表 |
| 分库分表 | 500 万行或 2GB |
| join 表数 | ≤ 3 |
| in 元素个数 | ≤ 1000 |
| varchar 索引长度 | 约 20（区分度可达 90%） |
| explain type | 至少 range，要求 ref，最好 consts |
| tcp_fin_timeout | 30 秒（默认 240 秒） |
| 二方库起始版本 | 1.0.0 |
| UML 必画的阈值 | 用例 > 5 个 → 用例图；状态 > 3 个 → 状态图；调用链对象 > 3 个 → 时序图；模型类 > 5 个 → 类图；协作对象 > 2 个 → 活动图 |

## 异常与日志按层处理
- DAO 层 → 抛 DAOException，不打日志
- Manager 层 → 与 Service 同机部署时同 DAO 的做法；单独部署时同 Service 的做法
- Service 层 → 打 error 日志，带上参数
- Web 层 → 不往上抛，跳转到友好的错误页
- 开放接口层 → 返回错误码和错误信息

## 识别问题的信号
- 看到 `Executors.new...` → 存在 OOM 风险
- 看到 `static SimpleDateFormat` → 线程不安全
- 看到 `ThreadLocal.set` 却找不到对应的 `remove` → 有泄漏
- 看到 `catch (Exception e) {}` 是空的 → 异常被吞掉
- 看到 finally 里有 return → 返回值被覆盖
- 看到 `Boolean isXxx` 定义在 POJO 里 → 会有序列化问题
- 看到 `new BigDecimal(0.1)` → 精度损失
- 看到 `"YYYY"` → 跨年那一周显示错误的年份
- 看到 `select *`、`${}`、`like '%x'` → 有 SQL 性能或安全问题
- 看到 `foreach` 循环里调用 `list.remove` → 会抛 ConcurrentModificationException
- 看到 `Collections.emptyList()` 被返回给调用方 → 调用方如果 add 会抛异常
