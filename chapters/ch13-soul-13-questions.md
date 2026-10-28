# 第 13 章：《Java开发手册（泰山版）灵魂13问》—— 规约背后的“为什么”

## Core Idea
手册只给结论，这 13 篇文章（作者 Hollis）从 **JLS、JDK 源码、反编译结果和实测数据** 出发，解释 13 条高频规约为什么这样定。共同的套路是：**语法糖和默认值背后藏着隐式行为**（自动拆箱、`StringBuilder`、`Iterator`、无界队列、自动生成的 serialVersionUID），规约就是把这些隐式行为变成显式约束。

> 说明：原文中的手册条款多以截图呈现，下文“对应规约”以本 skill 各章记录的条款为准；文章正文的论证、代码和数据经提炼重写，不是原文。

## 速览
| 问 | 题目 | 对应规约 | 一句话根因 |
|---|---|---|---|
| 1 | 三目运算符的空指针问题 | ch05 泰 8.4【强制】 | 第二、三操作数类型对齐时编译器插入 `xxxValue()` 拆箱 |
| 2 | 为什么建议初始化 HashMap 容量 | ch03 第 14 条【推荐】 | 超过 threshold 就成倍扩容并 rehash |
| 3 | HashMap 初始容量设多少合适 | ch03 第 14 条【推荐】 | JDK 只取 ≥ 传入值的 2 的幂，不考虑负载因子 |
| 4 | 为什么禁止 Executors 创建线程池 | ch04 第 4 条【强制】 | 无界 `LinkedBlockingQueue` 或 `Integer.MAX_VALUE` 个线程 → OOM |
| 5 | 为什么谨慎使用 subList | ch03 第 2、5 条【强制】 | 返回的是原 List 的视图 `SubList`，共享数据和 modCount |
| 6 | 为什么循环中不用 `+` 拼接字符串 | ch02 4.21【推荐】 | `+` 是语法糖，每轮循环都 new 一个 StringBuilder |
| 7 | 为什么禁止在 foreach 中 remove/add | ch03 第 11 条【强制】 | foreach 是 Iterator 语法糖，`modCount != expectedModCount` 触发 fail-fast |
| 8 | 为什么禁止直接使用 Log4j/Logback API | ch06 2.1【强制】 | 直接依赖实现会强耦合，门面模式让实现可替换 |
| 9 | 为什么禁止 static SimpleDateFormat | ch04 第 5 条【强制】 | format/parse 共用成员变量 `calendar` |
| 10 | 为什么禁止用 isSuccess 作变量名 | ch01 1.8【强制】 | JavaBeans getter 推导规则 vs 字段反射，序列化框架结果不一致 |
| 11 | 为什么禁止修改 serialVersionUID | ch02 4.13【强制】 | 反序列化时校验版本号，不一致抛 InvalidClassException |
| 12 | 为什么谨慎使用继承 | ch10 组合优于继承【推荐】（泰山版第 10 条） | 继承被当成复用手段，长期滥用导致维护成本高 |
| 13 | 为什么用 count(*) 而不用 count(列名/常量) | ch08 SQL 1【强制】 | 语义不同（是否计 NULL），且 MySQL 专门优化了 count(*) |

---

## 第 1 问：三目运算符的空指针问题
- **对应规约**：ch05 泰 8.4【强制】三目运算符的两个表达式在类型对齐时可能自动拆箱，抛 NPE。
- **根因/原理**：
  1. 自动装箱调用 `valueOf()`，自动拆箱调用 `xxxValue()`；包装对象为 null 时拆箱就会 NPE。
  2. JLS 15.25：第二、三操作数一个是基本类型 T、另一个是 T 的包装类型时，结果类型是**基本类型 T**，编译器会给包装操作数插入拆箱。Java SE 8 的 JLS 把条件表达式细分为布尔型、数值型、引用型三类并列表给出结果类型：**只有两边都是包装类型时，结果才是包装类型**。
  3. NPE 有两个来源：类型对齐时拆箱；或结果是包装类型但赋值给基本类型变量时拆箱。
  4. 反编译可以看到 `nullBoolean.booleanValue()`；作者在 JDK 6、8、14 上都复现了这个问题，与版本无关。
  5. 版本差异的特例：`Boolean b = map != null ? map.get("k") : false;` 在 JDK 8 以下 NPE，JDK 8 及以上返回 null。原因是 Java 8 把赋值上下文中的引用条件表达式当作 poly expression（JLS 15.25.3），按目标类型 Boolean 推断，两个操作数都转成 Boolean，不再拆箱。
- **反例**：
```java
boolean flag = true;
boolean simpleBoolean = false;
Boolean nullBoolean = null;
Boolean objectBoolean = Boolean.FALSE;
boolean x3 = flag ? nullBoolean : simpleBoolean;  // NPE：类型对齐时拆箱
boolean x4 = flag ? nullBoolean : objectBoolean;  // NPE：结果 Boolean(null) 赋给 boolean
Boolean x6 = flag ? nullBoolean : simpleBoolean;  // NPE：反编译为
// Boolean.valueOf(flag ? nullBoolean.booleanValue() : simpleBoolean)
```
- **结论**：让第二、三个表达式的类型一致，并且与接收变量类型一致；作者建议三者都用包装类型，再用单元测试覆盖 null 分支。

## 第 2 问：为什么建议初始化 HashMap 的容量
- **对应规约**：ch03 第 14 条【推荐】集合初始化时指定初始值大小。
- **根因/原理**：
  1. 关键字段：`size`、`loadFactor`（默认 0.75）、`capacity`（默认 16）、`threshold = capacity × loadFactor`。
  2. size 超过 threshold 就扩容，容量翻倍（16→32→64…），每次扩容都要重建 hash 表。
  3. 容量取 2 的幂，是为了用位与代替取模；0.75 = 3/4，乘以 2 的幂得到整数。
  4. 传入 initialCapacity 时，`tableSizeFor` 先减 1，再依次 `n |= n >>> 1/2/4/8/16`，最后加 1，得到“≥ 传入值的最小 2 的幂”。先减 1 是为了让 4 这类本身是 2 的幂的值不被翻倍成 8。
  5. JDK 1.8 在构造方法里确定容量，JDK 1.7 在第一次 put 时才确定，算法相同。
- **关键代码**（JDK 源码思路）：
```java
int n = cap - 1;
n |= n >>> 1;  n |= n >>> 2;  n |= n >>> 4;
n |= n >>> 8;  n |= n >>> 16;
return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
```
- **数据**：默认 HashMap 在 put 第 13 个元素时扩容（threshold 12→24）；JDK 1.7 下 put 1000 万个元素，不指定容量约 14.4s，指定一半约 11.9s，指定全量约 8.0s。
- **结论**：已知元素个数就在构造时指定容量；不要修改默认的 loadFactor。

## 第 3 问：HashMap 初始容量设多少合适
- **对应规约**：同第 2 问（ch03 第 14 条）。
- **根因/原理**：
  1. JDK 只机械地取 2 的幂，**不考虑 loadFactor**：`new HashMap(7)` 得到容量 8，但元素数达到 8 × 0.75 = 6 后就会扩容。
  2. 合理值参考 JDK 8 `putAll` 的实现（Guava 也采用）：`expectedSize / 0.75F + 1.0F`。
  3. Guava 的 `Maps.newHashMapWithExpectedSize`：expectedSize < 3 时返回 expectedSize + 1；小于 2^30 时按公式计算；否则返回 `Integer.MAX_VALUE`。
  4. 这是用内存换性能，元素很多时要权衡内存。
- **正例**：
```java
Map<String, String> map = Maps.newHashMapWithExpectedSize(7);       // Guava
int initialCapacity = (int) ((float) expectedSize / 0.75F + 1.0F);  // 手算
Map<String, String> m = new HashMap<>(initialCapacity);
```
- **结论**：不要直接 `new HashMap<>(元素个数)`；放 7 个元素时，7 / 0.75 + 1 ≈ 10，JDK 再取到 16，这样不会扩容。

## 第 4 问：为什么禁止用 Executors 创建线程池
- **对应规约**：ch04 第 4 条【强制】线程池不允许用 Executors 创建，要用 ThreadPoolExecutor。
- **根因/原理**：
  1. `newFixedThreadPool`、`newSingleThreadExecutor` 用的是**不指定容量的 `LinkedBlockingQueue`**，默认容量 `Integer.MAX_VALUE`，相当于无界，任务堆积会撑爆内存。
  2. `newCachedThreadPool`、`newScheduledThreadPool` 的最大线程数可能是 `Integer.MAX_VALUE`，大量创建线程同样会 OOM。
  3. 复现：`-Xmx8m` 下用 `newFixedThreadPool(15)` 循环提交 sleep 10s 的任务，抛 `OutOfMemoryError: GC overhead limit exceeded`，栈顶是 `LinkedBlockingQueue.offer`。
  4. `ArrayBlockingQueue` 必须指定容量；`LinkedBlockingQueue` 的容量可选，不指定就是无界。
  5. 换成有界队列后，队列满时抛 `RejectedExecutionException`。作者的观点是：抛 Exception 好过发生 Error。
- **正例**：
```java
ThreadFactory namedFactory = new ThreadFactoryBuilder()
        .setNameFormat("demo-pool-%d").build();          // Guava
ExecutorService pool = new ThreadPoolExecutor(5, 200,
        0L, TimeUnit.MILLISECONDS,
        new LinkedBlockingQueue<Runnable>(1024),         // 有界
        namedFactory, new ThreadPoolExecutor.AbortPolicy());
```
- **结论**：直接用 ThreadPoolExecutor 构造：有界队列 + 有名字的线程 + 显式拒绝策略。

## 第 5 问：为什么谨慎使用 ArrayList 的 subList
- **对应规约**：ch03 第 2 条【强制】subList 结果不能强转成 ArrayList；第 5 条【强制】subList 期间不要对原集合做结构性修改。
- **根因/原理**：
  1. `subList` 返回 `new SubList(this, 0, fromIndex, toIndex)`，是 ArrayList 的内部类，即**原列表的视图**；它只保存 parent、offset、size，以及当时的 `modCount`。
  2. SubList 与 ArrayList 没有继承关系，强转 ArrayList（或 LinkedList、Vector）抛 ClassCastException。
  3. 相互影响的三条规则：父子之间的非结构性修改（set）互相可见；子列表的结构性修改（add）会反映到父列表；父列表做了结构性修改后再操作子列表，`checkForComodification` 发现 modCount 不一致，抛 ConcurrentModificationException。
- **反例**：
```java
List<String> source = new ArrayList<>(Arrays.asList("H", "O", "L", "L", "I", "S"));
List<String> sub = source.subList(2, 5);  // [L, L, I]
sub.set(1, "666");                        // source 同步变化
source.add("666");                        // 父列表结构性修改
System.out.println(sub);                  // ConcurrentModificationException
```
- **结论**：用 List 接口接收，不强转；生成 subList 后不再对父列表做结构性修改；需要独立修改时先复制：`new ArrayList<>(sub)`，或用 `stream().skip(n).limit(m).collect(...)`。

## 第 6 问：为什么不在循环中用 `+` 拼接字符串
- **对应规约**：ch02 4.21【推荐】循环体内拼接字符串用 StringBuilder 的 append。
- **根因/原理**：
  1. String 不可变，任何“拼接”都会生成新对象。
  2. `+` 不是运算符重载，而是语法糖，反编译后是 `new StringBuilder().append(a).append(b).toString()`。
  3. `concat` 每次都 `Arrays.copyOf` 出一个新 char 数组再 new String；StringBuilder 内部是可变的 `char[]` 加 count，append 只在容量不够时扩容；StringBuffer 的 append 加了 `synchronized`。
  4. 循环里用 `+`，**每一轮都会 new 一个 StringBuilder** 再 toString，对象创建和内存浪费都很严重。
- **反编译对照**：
```java
String str = "hollis";
for (int i = 0; i < 50000; i++) {
    String s = String.valueOf(i);
    str += s;
    // 反编译：str = (new StringBuilder()).append(str).append(s).toString();
}
```
- **数据**（JDK 1.8，循环 5 万次，单位 ms）：StringBuilder 3 < StringBuffer 4 < concat 3623 < `+` 5119 < StringUtils.join 25726。
- **结论**：循环内用 StringBuilder；循环外的简单拼接直接用 `+`；需要线程安全时用 StringBuffer。

## 第 7 问：为什么禁止在 foreach 中 remove/add
- **对应规约**：ch03 第 11 条【强制】不要在 foreach 循环里 remove/add，删除用 Iterator。
- **根因/原理**：
  1. foreach 是语法糖，反编译后是 `Iterator` 的 `hasNext()`/`next()` 循环。
  2. `modCount` 是 ArrayList 的成员，记录结构性修改次数；`expectedModCount` 是迭代器 `Itr` 的成员，创建迭代器时从 modCount 复制，只有通过迭代器修改时才会同步更新。
  3. 调用集合自身的 `remove/add` 只改 modCount → 下一次 `next()` 中 `checkForComodification()` 发现两者不等，抛 ConcurrentModificationException。这就是 **fail-fast**，单线程下同样会触发。
  4. 异常不是在 remove 那一刻抛出的，而是在下一次 `next()` 时；所以删除倒数第二个元素时，`hasNext()` 直接返回 false，碰巧不报错。
- **正例**：
```java
Iterator<String> it = userNames.iterator();
while (it.hasNext()) {
    if ("Hollis".equals(it.next())) {
        it.remove();          // 同步更新 expectedModCount
    }
}
// 或 JDK8：userNames.removeIf("Hollis"::equals); / stream().filter(...).collect(...)
```
- **结论**：首选 `Iterator.remove()` 或 stream 过滤；普通 for 按下标删除不会触发 fail-fast，但会漏删；确定只删一个元素时可以 remove 后立刻 break；也可以改用 java.util.concurrent 下的 fail-safe 集合（遍历的是快照，看不到遍历期间的修改）。

## 第 8 问：为什么禁止直接使用 Log4j、Logback 的 API
- **对应规约**：ch06 2.1【强制】依赖 SLF4J、JCL 等日志门面的 API，不直接使用日志系统的 API。
- **根因/原理**：
  1. 常见实现有 j.u.l、Log4j、Logback、Log4j2，各自 API 不同；业务代码直接调用某个实现，就和它强耦合了。
  2. 日志门面应用的是**门面（外观）模式**：在应用和实现之间加一层统一接口，换实现只需要换 jar 包和配置，业务代码不动。原文的比喻：门面是服务员，实现是厨师。
  3. SLF4J 只是门面，真正输出日志的是 Log4j、Logback 等。它的优点：只保留 5 个级别（去掉 FATAL）；避免 `logger.error(e)` 只打 toString 的错误用法；支持 `{}` 占位符。
- **对照**：
```java
logger.debug("users: " + count + " " + list);              // 关闭 debug 也会拼接
if (logger.isDebugEnabled()) { logger.debug("users: " + count); }  // 啰嗦
logger.debug("There are now {} user accounts: {}", count, list);   // 推荐
```
- **结论**：业务代码只出现门面 API；最佳实践是“门面 + 实现”组合（如 SLF4J + Logback/Log4j）；用 `{}` 占位符。

## 第 9 问：为什么禁止把 SimpleDateFormat 定义成 static 变量
- **对应规约**：ch04 第 5 条【强制】SimpleDateFormat 线程不安全，一般不要定义为 static 变量。
- **根因/原理**：
  1. JDK 文档明确说 Date formats are not synchronized，建议每个线程各用一个实例。
  2. `format()` 把时间暂存在成员变量 `calendar` 里：线程 1 `calendar.setTime(11-11)` 后还没 `getTime()`，线程 2 就 `setTime(12-12)`，线程 1 拿到的是线程 2 的值。`parse()` 也有同样的问题。
  3. 复现：static 实例 + 线程池执行 100 个“当天加 i 天”的任务，结果放入 Set 去重，期望 100 个，实际少于 100 个。
- **正例**：
```java
private static final ThreadLocal<SimpleDateFormat> SDF =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
String s = SDF.get().format(new Date());
// JDK8+ 首选：DateTimeFormatter（不可变、线程安全）
String t = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
```
- **结论**：JDK8 用 DateTimeFormatter 搭配 LocalDateTime；低版本三选一：ThreadLocal（每个线程一个实例）、局部变量（频繁创建对象）、`synchronized` 只锁住 format 那一行。

## 第 10 问：为什么禁止用 isSuccess 作为变量名
- **对应规约**：ch01 1.8【强制】POJO 中的布尔变量不加 is 前缀；ch02 4.11【强制】POJO 属性和 RPC 返回值用包装类型。
- **根因/原理**：
  1. JavaBeans 规范：普通属性用 `getXxx()`；基本类型 boolean 属性用 `isXxx()`。字段 `isSuccess` 按规范 getter 应叫 `isIsSuccess()`，但 IDE 生成的是 `isSuccess()`，和字段 `success` 生成的 getter 同名。
  2. fastjson、jackson 遍历 **getter**，按规则推导属性名，`isSuccess()` 被推成 `success`；Gson 遍历**字段**，输出 `isSuccess`。
  3. 混用框架时：fastjson 序列化得到 `{"success":true}`，Gson 反序列化时找不到 `success` 字段，`isSuccess` 取默认值 false，结果和真实值相反。
  4. 包装类型同理：远程定价接口异常时，`Double` 返回 null，计算立即报错、问题暴露；`double` 返回 0.0，就会按 0 元照常扣费，异常被掩盖。
- **反例**：
```java
class Model3 implements Serializable {
    private boolean isSuccess;
    public boolean isSuccess() { return isSuccess; }
    public void setSuccess(boolean success) { isSuccess = success; }
}
// fastjson: {"success":true}   jackson: {"success":true}   Gson: {"isSuccess":true}
// gson.fromJson(JSON.toJSONString(model3), Model3.class) → isSuccess=false
```
- **结论**：布尔字段命名为 `success`，类型用 `Boolean`；这样在任何序列化框架下结果都一致。

## 第 11 问：为什么禁止修改 serialVersionUID
- **对应规约**：ch02 4.13【强制】序列化类新增属性时不改 serialVersionUID；完全不兼容的升级才修改。
- **根因/原理**：
  1. 反序列化时，`ObjectStreamClass.initNonProxy` 会比较字节流中的 serialVersionUID 和本地类的值，不一致抛 `InvalidClassException: local class incompatible`。
  2. 不显式声明时，JVM 用 `computeDefaultSUID` 根据类结构计算一个值；类一变（比如加一个字段），这个值就变，旧数据随之无法反序列化。
  3. 相关机制：Serializable 是标记接口；transient 字段不参与序列化；类里定义了 `writeObject/readObject` 就走自定义序列化（如 ArrayList）。
- **反例**：
```java
// 用 serialVersionUID = 1L 序列化到文件后，把类改成 2L 再读取
class User1 implements Serializable {
    private static final long serialVersionUID = 2L;   // 原为 1L
    private String name;
}
User1 u = (User1) ois.readObject();
// InvalidClassException: stream classdesc serialVersionUID = 1, local class serialVersionUID = 2
```
- **结论**：实现 Serializable 就显式声明 `private static final long serialVersionUID`；兼容升级时不改，只有完全不兼容时才改；在 IDEA 中开启 “Serializable class without serialVersionUID” 检查。

## 第 12 问：为什么谨慎使用继承
- **对应规约**：ch10 规约速查 8–12 中的“组合优于继承”【推荐】（泰山版设计规约第 10 条）：谨慎使用继承的方式扩展，优先使用聚合/组合；不得已使用继承时必须符合里氏代换原则。
- **根因/原理**（本次提取的文本只有这篇的开头，下面只整理已有内容）：
  1. 很多人把继承当作复用代码的默认手段，作者认为这是错的：**长期大量使用继承会带来很高的维护成本**。
  2. 复用用得好能提高效率，被滥用就会产生难以维护的代码。
  3. Java 代码复用有三种形式：**继承、组合、代理**。
  4. 继承与组合的具体对比，原文在提取文本中缺失，这里不做补充。
- **正例**（根据规约写的示意）：
```java
// 组合：复用 List 的能力，但不暴露、也不受其所有方法约束
public class OrderBag {
    private final List<Order> orders = new ArrayList<>();
    public void add(Order o) { orders.add(o); }
}
// 反例：class OrderBag extends ArrayList<Order>  —— 调用方可以绕过业务约束直接 remove/clear
```
- **结论**：做扩展时优先用组合；只有 is-a 关系成立、子类能完全替换父类时才用继承（ch10）。

## 第 13 问：为什么用 count(*) 而不用 count(列名) 或 count(常量)
- **对应规约**：ch08 SQL 1【强制】不要用 count(列名) 或 count(常量) 替代 count(*)。
- **根因/原理**：
  1. **语义不同**：`COUNT(expr)` 统计 expr 不为 NULL 的行数。常量和 `*` 不可能为 NULL，所以 `COUNT(1)` 和 `COUNT(*)` 统计的是总行数；`COUNT(列名)` 统计的是该列非 NULL 的行数。
  2. `COUNT(*)` 是 SQL92 定义的标准写法，MySQL 对它做了专门优化。
  3. MyISAM 只有表锁，会单独记录总行数，不带 WHERE 的 `COUNT(*)` 直接返回；InnoDB 支持事务和行锁，行数随事务可见性变化，只能扫描。
  4. MySQL 8.0.13 起，InnoDB 对不带 WHERE / GROUP BY 的 `COUNT(*)` 优先选**最小的非聚簇索引**扫描（叶子节点只存主键，比聚簇索引小）。
  5. 官方文档说明 InnoDB 对 `COUNT(*)` 和 `COUNT(1)` 的处理方式相同、性能没有差别；`COUNT(列名)` 还要逐行判断是否为 NULL，更慢。
- **示例**：
```sql
-- 7 行数据：id 有 3 个非 NULL，id2 有 2 个非 NULL
SELECT COUNT(*), COUNT(id), COUNT(id2) FROM t;   -- 7, 3, 2
```
- **结论**：统计行数一律用 `COUNT(*)`；不要以为 `COUNT(1)` 更快（以上结论只针对 MySQL）；`COUNT(列名)` 只用于“统计该列非 NULL 的行数”。

---

## Mental Models
- **看到语法糖就反编译**：foreach、`+`、自动拆装箱、三目运算符的行为，都要看编译器实际生成的代码。
- **默认值往往是隐患**：默认无界队列、默认生成的 serialVersionUID、基本类型默认 0/false、JDK 不考虑负载因子的容量计算，都需要显式指定。
- **视图与共享状态**：subList、static SimpleDateFormat 的问题本质相同——看似独立的对象，实际共享底层状态。
- **契约在边界上验证**：isSuccess 和 serialVersionUID 的问题都在序列化边界上暴露，本地测试往往发现不了。

## Key Takeaways
1. 三目运算符两侧类型保持一致，避免包装类型与基本类型混用。
2. HashMap 初始容量 = 期望元素数 / 0.75 + 1，或用 Guava `newHashMapWithExpectedSize`。
3. 线程池用 ThreadPoolExecutor 加有界队列；subList 要修改就先复制；遍历时删除用 Iterator。
4. 循环内用 StringBuilder 拼接；日志只依赖门面 API 并使用 `{}` 占位符。
5. JDK8 用 DateTimeFormatter；布尔字段不加 is 并用包装类型；显式声明且不随意修改 serialVersionUID。
6. 统计行数用 `COUNT(*)`；扩展时优先组合。

## Connects To
- **ch01 / ch02**：第 6、10、11 问对应的命名与 OOP 规约。
- **ch03 / ch04**：第 2、3、4、5、7、9 问对应的集合与并发规约。
- **ch05 / ch06**：第 1、8 问（三目运算符 NPE、日志门面）。
- **ch08 / ch10**：第 13、12 问（count(*)、谨慎继承）。
- **ch11**：日期时间规约与第 9 问互为补充。
