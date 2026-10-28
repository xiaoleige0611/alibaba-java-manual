# 第 4 章：编程规约 —（六）并发处理

## Core Idea
并发代码的事故大多来自**资源失控**（无界队列、无限创建线程导致 OOM）、**状态泄漏**（ThreadLocal 不清理、SimpleDateFormat 共享）和**锁的误用**（lock 写在 try 里面、加锁顺序不一致）。手册的应对办法是：线程池必须显式配置、线程要命名、锁要按固定模板写、锁的粒度尽量小。

## Frameworks Introduced
- **禁用 Executors，必须用 ThreadPoolExecutor**：
  - `FixedThreadPool` / `SingleThreadPool`：请求队列长度为 `Integer.MAX_VALUE`，请求堆积会导致 OOM。
  - `CachedThreadPool`：允许创建的线程数为 `Integer.MAX_VALUE`，线程过多会导致 OOM。
  - How：显式指定核心线程数、最大线程数、**有界队列**、带名字的 ThreadFactory、拒绝策略。
- **锁粒度三原则**：能用无锁数据结构就不用锁；能锁代码块就不锁整个方法；能用对象锁就不用类锁。锁内不要调用 RPC。
- **阻塞加锁模板**：`lock.lock()` 写在 try **之外**，并且 lock 与 try 之间不能有任何可能抛异常的调用；在 finally 中 unlock。
- **尝试加锁模板**：先判断 `tryLock()` 是否成功，成功后再进入 try/finally。
- **一致的加锁顺序**：线程一按 A→B→C 加锁，线程二也必须按 A→B→C，否则可能死锁。
- **乐观锁与悲观锁的选择**：冲突概率 < 20% 用乐观锁（version 字段），且**重试次数不少于 3 次**；否则用悲观锁；**资金相关的金融敏感信息用悲观锁**。
- **DCL（双重检查锁）**：延迟初始化的目标属性要声明为 `volatile`（JDK5+）。

## 规约速查
> 泰山版本节（七）并发处理仍为 19 条，与华山版条数一致；版本历史点名修订了第 9 条“阻塞等待锁”的描述（见下）。

| # | 级别 | 规则 |
|---|---|---|
| 1 | 强制 | 获取单例对象要保证线程安全，单例里的方法也要线程安全 |
| 2 | 强制 | 创建线程或线程池时指定有意义的名称（方便 jstack 排查） |
| 3 | 强制 | 线程资源必须由线程池提供，不允许在应用中显式创建线程 |
| 4 | 强制 | 不用 Executors，用 ThreadPoolExecutor。原理见 ch13 第 4 问 |
| 5 | 强制 | SimpleDateFormat 线程不安全，不要定义成 static；必须定义成 static 时要加锁或用 DateUtils，亦推荐 ThreadLocal；JDK8 用 DateTimeFormatter。原理见 ch13 第 9 问 |
| 6 | 强制 | 自定义的 ThreadLocal 必须在 try-finally 中 remove（线程池会复用线程） |
| 7 | 强制 | 高并发时考虑锁的性能损耗（锁粒度三原则） |
| 8 | 强制 | 对多个资源加锁时保持一致的顺序 |
| 9 | 强制 | 阻塞加锁时 lock 写在 try 之外，且 lock 与 try 之间不能有可能抛异常的调用（泰山版修订描述，见下） |
| 10 | 强制 | 尝试加锁时先判断是否持有锁 |
| 11 | 强制 | 并发修改同一条记录要加锁：应用层锁、缓存锁，或数据库乐观锁（version） |
| 12 | 强制 | 多个定时任务用 ScheduledExecutorService，不用 Timer（Timer 中一个任务抛异常，其他任务全部终止） |
| 13 | 推荐 | 资金相关数据用悲观锁 |
| 14 | 推荐 | CountDownLatch：每个线程退出前都要调用 countDown，注意 catch 异常；子线程的异常在主线程 catch 不到 |
| 15 | 推荐 | 不要让多个线程共用一个 Random 实例（竞争 seed 导致性能下降），JDK7+ 用 ThreadLocalRandom |
| 16 | 推荐 | DCL 的目标属性加 volatile |
| 17 | 参考 | volatile 只能解决一写多读的场景；count++ 用 AtomicInteger，JDK8 用 LongAdder |
| 18 | 参考 | HashMap 在高并发 resize 时可能出现死链，导致 CPU 飙升 |
| 19 | 参考 | ThreadLocal 用 static 修饰；它无法解决共享对象的更新问题 |

## 泰山版：阻塞等待锁（第 9 条）的三条说明
1. lock 与 try 之间的调用抛异常 → 锁已加上却走不到 finally，**永远无法解锁**，其他线程拿不到锁。
2. lock 写在 try 内 → 前面的代码抛异常时，finally 对未加锁的对象 unlock，经 AQS `tryRelease` 抛 IllegalMonitorStateException。
3. `lock()` 方法本身也可能抛 unchecked 异常，后果同第 2 条。

## Key Concepts
- **AQS / tryRelease**：unlock 对未持有的锁调用时会抛 IllegalMonitorStateException。
- **ThreadLocal 泄漏**：线程池复用线程，上一个请求留下的值会污染下一个请求，并造成内存泄漏。
- **乐观锁**：用 `version` 字段作为更新依据，`update ... where version = ?`。
- **LongAdder**：分段累加，在高竞争下比 AtomicLong 的重试次数少，性能更好。
- **volatile**：保证可见性，不保证复合操作的原子性。

## Mental Models
- **无界 = 隐患**：任何“无限”的配置（队列长度、线程数）都是迟早会爆的 OOM。
- **锁是借来的，必须还**：把 lock 放在 try 外面，就保证了“加锁成功 ⇒ 一定会 unlock”，“没加上锁 ⇒ 不会 unlock”。
- **ThreadLocal 要成对使用**：set 与 remove 一一对应，就像打开文件后要关闭。
- **把冲突率当作选锁的开关**：20% 是分界线；涉及资金一律用悲观锁。

## Anti-patterns
- `Executors.newFixedThreadPool(n)`：请求堆积到 OOM。
- `private static final SimpleDateFormat sdf`：多线程下解析和格式化结果错乱。
- 把 `lock.lock()` 写在 try 里面：lock 之前的代码抛异常时，finally 会对未持有的锁调用 unlock，抛 IllegalMonitorStateException。
- 用 Timer 跑多个定时任务：一个任务异常，其他任务全部停止。
- 不加 volatile 的 DCL：可能拿到未初始化完成的对象。
- 在锁里调用 RPC：锁的持有时间被远程调用拖长。

## Code Examples
```java
// 带名字的线程工厂
public class UserThreadFactory implements ThreadFactory {
    private final String namePrefix;
    private final AtomicInteger nextId = new AtomicInteger(1);
    UserThreadFactory(String whatFeaturOfGroup) {
        namePrefix = "From UserThreadFactory's " + whatFeaturOfGroup + "-Worker-";
    }
    @Override
    public Thread newThread(Runnable task) {
        String name = namePrefix + nextId.getAndIncrement();
        return new Thread(null, task, name, 0, false);
    }
}

// 阻塞加锁：正例
Lock lock = new XxxLock();
lock.lock();
try {
    doSomething();
    doOthers();
} finally {
    lock.unlock();
}

// 尝试加锁：正例
boolean isLocked = lock.tryLock();
if (isLocked) {
    try {
        doSomething();
    } finally {
        lock.unlock();
    }
}

// ThreadLocal 回收
objectThreadLocal.set(userInfo);
try {
    // ...
} finally {
    objectThreadLocal.remove();
}

// SimpleDateFormat 的线程安全写法（JDK8 优先用 DateTimeFormatter）
private static final ThreadLocal<DateFormat> df = new ThreadLocal<DateFormat>() {
    @Override
    protected DateFormat initialValue() {
        return new SimpleDateFormat("yyyy-MM-dd");
    }
};
```
- **What it demonstrates**：线程命名、两种加锁模板、ThreadLocal 回收。

## Worked Example
需求：给一个导出服务配置线程池。按手册推导：
```java
ThreadPoolExecutor exportPool = new ThreadPoolExecutor(
    4,                                   // 核心线程数
    8,                                   // 最大线程数，有上限
    60L, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(200),       // 有界队列，避免 Fixed/Single 线程池的 OOM 问题
    new UserThreadFactory("export"),     // 有意义的线程名（第 2 条）
    new ThreadPoolExecutor.CallerRunsPolicy());   // 明确拒绝策略
```
- 任务里如果用到 ThreadLocal 存放用户上下文，必须在 finally 中 remove（第 6 条）。
- 汇总多个子任务的结果用 CountDownLatch：每个子任务在 finally 中 countDown，主线程 `await(timeout)`（第 14 条）。
- 如果要更新导出记录的状态（冲突概率低）→ 用 version 乐观锁，重试不少于 3 次（第 11 条）；如果更新的是账户余额 → 用悲观锁（第 13 条）。

## Key Takeaways
1. 线程池用 ThreadPoolExecutor 显式配置：有界队列、带名字的线程、拒绝策略。
2. `lock()` 写在 try 外面，`unlock()` 写在 finally 里面；tryLock 要先判断结果。
3. ThreadLocal 用完必须 remove；SimpleDateFormat 不能裸着共享。
4. 冲突率 < 20% 用乐观锁（重试 ≥ 3 次），资金数据用悲观锁。
5. 多资源加锁顺序要一致；锁粒度尽量小；锁里不调用 RPC。
6. 计数用 AtomicInteger 或 LongAdder；volatile 只适用于一写多读。

## Connects To
- **ch03**：并发场景下 Iterator 要加锁，HashMap 有死链风险。
- **ch05**：高并发下不要用“等于”判断作为终止条件。
- **ch09**：服务器层面的 OOM 防护（HeapDumpOnOutOfMemoryError）。
- **ch11**：日期时间规约（DateTimeFormatter、闰年）。
- **ch13**：第 4 问（Executors 的 OOM 根因）、第 9 问（SimpleDateFormat 共享 Calendar 的线程安全问题）。
