# 第 6 章：异常日志 —（一）异常处理 +（二）日志规约

## Core Idea
异常是为**意外情况**准备的，不是用来做流程控制的；捕获异常是为了**处理**它，不能吞掉。日志的价值在于**还原案发现场**：要有现场参数和异常堆栈，用 SLF4J 门面，用占位符，按级别开关，不重复打印、不刷屏。

## Frameworks Introduced
- **预检查优先于 catch**：能通过预检查规避的 RuntimeException（NPE、IndexOutOfBoundsException）不要用 catch 处理；无法预检查的（如 NumberFormatException）才 catch。
- **稳定代码与非稳定代码**：只对非稳定代码 try-catch，并且按异常类型分别处理；对大段代码整体 try-catch 是不负责任的做法。
- **错误码 / 异常 / Result 三分法**（【参考】）：
  | 场景 | 方式 |
  |---|---|
  | 公司外部的 http/api 开放接口 | **错误码** |
  | 应用内部 | 抛异常 |
  | 跨应用的 RPC 调用 | **Result**（`isSuccess()` + 错误码 + 错误简短信息） |
  - Why Result：调用方没捕获到异常就会产生运行时错误；不带栈信息的异常对调用方帮助不大，带栈信息又会在频繁出错时增加序列化和传输的开销。
- **NPE 六大场景**：返回类型是基本类型却 return 包装类型对象（自动拆箱）；数据库查询结果为 null；集合 isNotEmpty 但取出的元素为 null；远程调用的返回对象；Session 中取出的数据；级联调用 `obj.getA().getB().getC()`。→ 用 JDK8 的 Optional 防护。
- **防止 NPE 是调用者的责任**：即使被调用方承诺返回空集合，调用方也要考虑远程调用失败、序列化失败等返回 null 的情况。
- **日志的两类必备信息**：案发现场信息（参数、对象的 toString）+ 异常堆栈。
- **日志级别的用法**：error 只记录系统逻辑出错、异常或重要的错误信息；warn 可以记录用户输入参数错误（避免频繁报警）；生产环境禁止输出 debug；info 有选择地输出。

## 规约速查
> 编号沿用华山版；标 `泰 x.y` 的行是泰山版（v1.6.0）新增条款，x.y 为泰山版编号；泰山版异常日志新增（一）错误码（见 ch12），异常处理变为（二）、日志变为（三）（泰山版插入新条款后，后续编号整体后移）。

| # | 级别 | 规则 |
|---|---|---|
| 1.1 | 强制 | 能预检查的 RuntimeException 不要 catch |
| 1.2 | 强制 | 不用异常做流程控制或条件控制 |
| 1.3 | 强制 | 区分稳定代码和非稳定代码；按异常类型分别处理 |
| 1.4 | 强制 | 捕获后必须处理，否则往上抛；最外层必须转化成用户能理解的内容 |
| 1.5 | 强制 | 事务代码中 catch 了异常，如果需要回滚，要手动回滚 |
| 1.6 | 强制 | finally 中关闭资源和流，关闭操作本身也要 try-catch；JDK7+ 用 try-with-resources |
| 1.7 | 强制 | **finally 中不要 return** |
| 1.8 | 强制 | catch 的异常与抛出的异常要完全匹配，或者是其父类 |
| 1.9 | 强制 | 调用 RPC、二方包、动态生成类的方法时，用 **Throwable** 捕获（防 NoSuchMethodError） |
| 1.10 | 推荐 | 方法可以返回 null，但必须注释说明什么情况下返回 null |
| 1.11 | 推荐 | 注意 NPE 六大场景（泰山版另在控制语句中新增三目运算符拆箱 NPE，见 ch05、ch13 第 1 问） |
| 1.12 | 推荐 | 区分 checked 和 unchecked 异常；不直接抛 `new RuntimeException()`，更不能抛 Exception 或 Throwable；使用有业务含义的自定义异常（DAOException、ServiceException） |
| 1.13 | 参考 | 错误码 / 异常 / Result 三分法 |
| 1.14 | 参考 | DRY 原则：重复的校验逻辑抽成 `private boolean checkParam(DTO dto)` |
| 2.1 | 强制 | 使用 **SLF4J**（或 JCL）门面 API，不要直接使用 Log4j、Logback 的 API。原理见 ch13 第 8 问 |
| 2.2 | 强制 | 日志至少保留 **15 天**（有些异常按“周”发生）；当天日志 `应用名.log`，历史日志 `{logname}.log.{yyyy-MM-dd}`。注：泰山版原文未见“敏感操作日志 ≥ 6 个月”的表述 |
| 2.3 | 强制 | 扩展日志命名：`appName_logType_logName.log`（logType 如 stats、monitor、access） |
| 2.4 | 强制 | 日志字符串拼接用 `{}` 占位符 |
| 2.5 | 强制 | trace/debug/info 级别的日志要先判断级别开关 |
| 2.6 | 强制 | 避免重复打印：在 log4j.xml 中设置 `additivity=false` |
| 泰 3.7 | 强制 | **生产环境禁止用 System.out / System.err 输出日志，禁止 `e.printStackTrace()`**（标准输出文件只在容器重启时滚动，易超过文件大小限制） |
| 2.7 | 强制 | 异常日志要包含现场信息和堆栈：`logger.error(参数 + "_" + e.getMessage(), e)` |
| 泰 3.9 | 强制 | **打日志时禁止直接用 JSON 工具把对象转成 String**（被重写的 getter 可能抛异常，打日志反而中断业务）；只打业务属性或调用 toString() |
| 2.8 | 推荐 | 生产环境禁止 debug；用 warn 记录上线初期的观察信息时注意日志量，并及时删除 |
| 2.9 | 推荐 | 用户输入参数错误记 warn，不要记 error |
| 2.10 | 推荐 | 错误信息尽量用英文描述，英文说不清时用中文；国际化团队或海外部署的服务器用全英文（泰山版写在说明中，未单独标【强制】） |

## Key Concepts
- **NoSuchMethodError**：二方包版本仲裁引入了非预期版本，或者 ASM 等字节码框架改了方法签名，编译期正常，运行期报错；它是 Error，catch Exception 捕获不到。
- **try-with-resources**：JDK7 引入的自动关闭资源语法。
- **门面模式（SLF4J）**：业务代码只依赖接口，日志实现可以随时替换。
- **additivity**：logger 是否把日志事件传递给父 logger。

## Mental Models
- **异常是给意外准备的保险，不是日常的交通工具**：用异常做流程控制既慢，又让代码意图含糊。
- **抛出的是绣球，接到的却是铅球**：catch 的类型必须能覆盖实际抛出的类型。
- **写日志前问三个问题**：这条日志真的有人看吗？看到它能做什么？能不能帮助排查问题？
- **DAO 只负责往上抛，Service 负责记录**：同一台机器上不要把同一个异常记两遍（见 ch09 的分层异常处理）。

## Anti-patterns
- `try { obj.method(); } catch (NullPointerException e) {}`：应该改成 `if (obj != null)`。
- catch 之后什么也不做：异常被吞掉，线上问题无从排查。
- 在 finally 中 `return ++x`：try 中的返回值被丢弃（原文示例返回 2 而不是 1）。
- 调用二方包时只 catch Exception：NoSuchMethodError 直接打穿。
- `logger.debug("id: " + id)`：即使 debug 关闭，字符串拼接也会执行。
- 每个参数错误都记 error：报警泛滥。

## Code Examples
```java
// 反例：finally 中 return
private int x = 0;
public int checkReturn() {
    try {
        return ++x;      // x 等于 1，这里不会立刻返回
    } finally {
        return ++x;      // 实际返回 2
    }
}

// SLF4J 标准写法
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
private static final Logger logger = LoggerFactory.getLogger(Test.class);

// 占位符 + 级别开关
if (logger.isDebugEnabled()) {
    logger.debug("Current ID is: {} and name is: {}", id, getName());
}

// 异常日志：现场信息 + 堆栈
logger.error(各类参数或者对象 toString() + "_" + e.getMessage(), e);

// 避免重复打印
<logger name="com.taobao.dubbo.config" additivity="false">
```
- **What it demonstrates**：finally 的执行顺序陷阱；日志写法的标准模板。

## Worked Example
设计一个跨应用的 RPC 接口 `PayService.pay(PayRequest)` 的异常与日志方案：
1. 返回类型用 `Result<PayResultDTO>`，包含 `isSuccess()`、`errorCode`、`errorMsg`（1.13）。
2. 服务端内部：DAO 层 `catch (Exception e) { throw new DAOException(e); }`，不打日志；Service 层 catch 后执行 `logger.error("pay failed, req={}", request, e)`，然后转成 Result 返回（ch09 分层异常处理）。
3. 调用方：`Result r = payService.pay(req);` 外面用 `catch (Throwable t)` 包住，防止二方包版本冲突导致的 NoSuchMethodError（1.9）；拿到 r 先判空（1.11 远程调用场景）。
4. 用户传入的金额格式错误 → 记 warn 级别日志，不记 error（2.9）。
5. 支付日志单独存放：`pay_monitor_payResult.log`（2.3），至少保留 15 天（2.2）；打印 request 时用 toString，不用 JSON 工具（泰 3.9）。
6. 对外返回的错误码按 ch12 的五位规则，如参数格式错误用 A0400 段，第三方支付超时用 C0200 段并带上原始错误码。
7. 如果涉及事务，catch 之后要手动回滚（1.5）。

## Key Takeaways
1. 能预检查的就不要 catch；异常不做流程控制；捕获了就必须处理，或者往上抛。
2. finally 负责关闭资源、不要 return；JDK7+ 用 try-with-resources。
3. 调用 RPC 或二方包时 catch Throwable；对外接口用错误码，跨应用 RPC 用 Result。
4. 防止 NPE 是调用者的责任；牢记六大 NPE 场景，用 Optional 防护。
5. 日志：SLF4J 加 `{}` 占位符，debug/info 先判断级别，异常日志带现场信息和堆栈，`additivity=false`。
6. 日志至少保留 15 天；参数错误记 warn，不记 error。
7. （泰山）生产环境不用 System.out 和 printStackTrace；不用 JSON 工具序列化对象打日志。

## Connects To
- **ch02**：自动拆箱导致的 NPE；POJO 的 toString 用于打印现场信息。
- **ch09**：分层异常处理规约（DAO / Service / Web / 开放接口各层的职责）。
- **ch12**：泰山版错误码统一方案（A/B/C 来源 + 四位编号）。
- **ch13 第 8 问**：门面模式与日志框架的选择。
- **ch08**：`@Transactional` 不要滥用，回滚需要综合考虑缓存、消息等。
