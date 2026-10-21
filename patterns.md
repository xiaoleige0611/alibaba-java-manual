# 技法与模式

## 卫语句（Guard Clause）
**When to use**：if-else 超过 3 层，或者异常分支很多时。
**How**：在方法开头依次判断失败、异常、中断条件，直接 return；剩下的代码就是主流程。也可以改用策略模式或状态模式。
**Trade-offs**：方法会有多个出口；但嵌套更少，也更容易测试。(Ch 5)

## 阻塞加锁模板
**When to use**：使用 `Lock.lock()` 时。
**How**：`lock.lock();` 写在 try 之外，并且紧挨着 try（中间不能有可能抛异常的调用）；业务代码写在 try 里；在 finally 中 `lock.unlock()`。
**Trade-offs**：没有额外开销，只是写法上的约束；把 lock 写进 try 会导致对未持有的锁调用 unlock，抛 IllegalMonitorStateException。(Ch 4)

## 尝试加锁模板
**When to use**：使用 `tryLock()` 时。
**How**：`if (lock.tryLock()) { try { ... } finally { lock.unlock(); } }`
**Trade-offs**：获取不到锁时要有降级逻辑。(Ch 4)

## 自定义线程池（替代 Executors）
**When to use**：任何需要线程的地方（不允许显式创建线程）。
**How**：用 `new ThreadPoolExecutor(core, max, keepAlive, unit, 有界队列, 带名字的 ThreadFactory, 拒绝策略)`。
**Trade-offs**：需要预估容量；但能避免无界队列或无限线程导致的 OOM，jstack 排查时也能看到线程名。(Ch 4)

## ThreadLocal 成对回收
**When to use**：在线程池中使用自定义的 ThreadLocal 时。
**How**：`set` 之后紧跟 `try { ... } finally { tl.remove(); }`；ThreadLocal 变量用 static 修饰。
**Trade-offs**：不回收会导致上下文串到下一个请求，并造成内存泄漏。(Ch 4)

## 乐观锁 / 悲观锁选择
**When to use**：并发修改同一条记录时。
**How**：冲突概率低于 20% → 用 version 乐观锁，重试不少于 3 次；冲突概率高或涉及资金 → 用悲观锁。
**Trade-offs**：乐观锁吞吐更高，但校验逻辑容易出漏洞；悲观锁更安全，但会阻塞。(Ch 4)

## 浮点数比较
**When to use**：需要判断浮点数是否相等时。
**How**：用误差范围判断 `Math.abs(a - b) < 1e-6f`；或者用 `new BigDecimal("x")` / `BigDecimal.valueOf(x)` 计算后再 equals。
**Trade-offs**：BigDecimal 精确但慢；误差范围法快，但要选合适的阈值。(Ch 2)

## 集合安全转换
**When to use**：使用 subList、Arrays.asList、toArray 时。
**How**：要修改视图时先复制一份 `new ArrayList<>(view)`；转数组用 `toArray(new T[0])`；遍历时删除用 Iterator。
**Trade-offs**：复制有少量开销，但能避免 ClassCastException、UnsupportedOperationException 和 ConcurrentModificationException。(Ch 3)

## Result 返回模式
**When to use**：跨应用的 RPC 接口。
**How**：返回 `Result<T>`，包含 `isSuccess()`、错误码和错误简短信息；对外 http 接口用错误码；应用内部抛异常。
**Trade-offs**：调用方必须检查 Result；但能避免异常在 RPC 边界上丢失，或产生过重的堆栈序列化。(Ch 6)

## 分层异常处理
**When to use**：多层架构中。
**How**：DAO 用 `throw new DAOException(e)`，不打日志；Service 记录日志并带上参数；Web 跳转到友好的错误页；开放接口返回错误码。
**Trade-offs**：职责清晰，同一个异常只记录一次日志。(Ch 9)

## 日志标准写法
**When to use**：所有日志输出。
**How**：用 SLF4J 的 `private static final Logger`，用 `{}` 占位符；debug/info 先用 `isXxxEnabled()` 判断；异常日志写成 `logger.error(现场信息 + "_" + e.getMessage(), e)`；配置 `additivity=false`。
**Trade-offs**：写法稍长，但性能好，便于排查。(Ch 6)

## 延迟关联分页
**When to use**：offset 很大的深分页。
**How**：`SELECT a.* FROM t a, (SELECT id FROM t WHERE 条件 LIMIT 100000,20) b WHERE a.id=b.id`
**Trade-offs**：SQL 写起来更复杂；也可以改成控制可翻的总页数。(Ch 8)

## 组合索引设计
**When to use**：多条件查询并且带排序时。
**How**：等号条件的列在前，区分度高的列在前，order by 的列放在最后；用 explain 确认 type 至少达到 range。
**Trade-offs**：索引越多写入越慢；既不要宁滥勿缺，也不要宁缺勿滥。(Ch 8)

## 依赖注入以支持可测
**When to use**：被测对象依赖外部服务或中间件时。
**How**：把依赖改成注入，测试时用 DI 框架注入内存实现或 Mock 实现。
**Trade-offs**：需要在设计阶段就考虑到；能换来可重复执行（AIR 中的 R）。(Ch 7)

## 隔离变化点（开闭 + 依赖倒置）
**When to use**：同一业务域内存在多种实现，且需求会变化（例如多种支付渠道）。
**How**：定义接口，每种实现一个类，调用方只依赖接口；新增实现时不修改已有代码。
**Trade-offs**：会多出一些类；但只有在真正存在变化点时才值得这样做。(Ch 10)
