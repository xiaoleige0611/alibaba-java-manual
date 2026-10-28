# 第 5 章：编程规约 —（七）控制语句 +（八）注释规约 +（九）其它

## Core Idea
控制语句的核心是**减少嵌套、让分支清晰**（卫语句、if-else 不超过 3 层、复杂条件赋值给有意义的布尔变量）；注释的核心是**说明意图并与代码保持同步**；“其它”一节收录了日期格式、随机数、正则预编译、模板输出等零散但高频的坑。

## Frameworks Introduced
- **卫语句（Guard Clause）**：先处理失败、异常、中断、退出等需要直接返回的情况，让方法有多个出口，以此消除嵌套。这是一种逆向思维。
  - When：if-else 超过 3 层（**【强制】不超过 3 层**），或者异常分支很多时。
  - 其他替代方案：策略模式、状态模式。
- **参数校验的取舍**：
  | 需要校验 | 不需要校验 |
  |---|---|
  | 调用频次低的方法 | 极有可能被循环调用的方法（要在方法说明里注明外部参数检查要求） |
  | 执行开销很大的方法 | 调用频度高的底层方法（同机部署的 DAO 层） |
  | 要求极高稳定性和可用性的方法 | private 方法，且能确定调用方已经检查过参数 |
  | 对外开放的接口（RPC/API/HTTP） | |
  | 敏感权限入口 | |
- **特殊注释标记**：`TODO`（标记人，标记时间，[预计处理时间]）表示功能尚未实现；`FIXME`（同上格式）表示代码有错误、不能工作，需要尽快修正。要定期扫描并清理这些标记。
- **日期格式符号**：`yyyy` 表示年（大写 `YYYY` 是 week-year，本周跨年时会返回下一年），`MM` 表示月，`mm` 表示分钟，`HH` 表示 24 小时制，`hh` 表示 12 小时制。

## 规约速查
> 编号沿用华山版；标 `泰 x.y` 的行是泰山版（v1.6.0）新增条款，x.y 为泰山版编号；泰山版中控制语句为（八）、注释为（九）、其它为（十）（泰山版插入新条款后，后续编号整体后移）。

| # | 级别 | 规则 |
|---|---|---|
| 7.1 | 强制 | switch 的每个 case 要用 break/continue/return 结束，或者注释说明会继续执行到哪个 case；必须有 default 且放在最后 |
| 7.2 | 强制 | switch 的变量是 String 且来自外部参数时，先判断 null（`switch(null)` 会抛 NPE） |
| 7.3 | 强制 | if/else/for/while/do 必须使用大括号，即使只有一行 |
| 泰 8.4 | 强制 | **三目运算符 `cond ? 表达式1 : 表达式2` 类型对齐时可能自动拆箱导致 NPE**：两个表达式中只要有一个是基本类型，或二者类型不一致（会拆箱并升级为范围更大的类型），就会触发拆箱。原理见 ch13 第 1 问 |
| 7.4 | 强制 | 高并发场景下不要用“等于”判断作为中断或退出条件，改用区间判断 |
| 泰 8.6 | 推荐 | 方法超过 10 行时，return / throw 等中断逻辑的右大括号后加一个空行 |
| 7.5 | 推荐 / 强制 | 少用 if-else，多用卫语句；if-else **不超过 3 层**（泰山版第 8.7 条整体为【推荐】，“请勿超过 3 层”写在说明里，未再单独标【强制】） |
| 7.6 | 推荐 | 复杂的条件判断结果赋值给有意义的布尔变量 |
| 7.7 | 推荐 | 不要在表达式（尤其是条件表达式）中插入赋值语句 |
| 7.8 | 推荐 | 定义对象、获取数据库连接、不必要的 try-catch 尽量移到循环体外 |
| 7.9 | 推荐 | 避免使用取反逻辑：写 `x < 628`，不写 `!(x >= 628)` |
| 7.10 | 推荐 | 批量操作的接口要做入参保护 |
| 8.1 | 强制 | 类、类属性、类方法的注释用 Javadoc `/** */`，不用 `//` |
| 8.2 | 强制 | 抽象方法（包括接口方法）的 Javadoc 除了返回值、参数、异常外，还要说明做什么、实现什么功能 |
| 8.3 | 强制 | 所有类都要写创建者和创建日期 |
| 8.4 | 强制 | 方法内的单行注释写在被注释语句的上方，用 `//`；多行注释用 `/* */` |
| 8.5 | 强制 | 所有枚举字段都要有注释 |
| 8.6 | 推荐 | 与其写“半吊子”英文注释，不如用中文说清楚；专有名词保持英文 |
| 8.7 | 推荐 | 修改代码的同时修改注释 |
| 泰 9.8 | 推荐 | 删除类中未使用的字段和方法、方法中未使用的参数声明和内部变量 |
| 8.8 | 参考 | 不要随便注释掉代码：要么在上方写明原因，要么直接删除 |
| 8.10 | 参考 | 命名和结构好的代码是自解释的，避免过多过滥的注释 |
| 9.1 | 强制 | 正则表达式要预编译，不要在方法体内调用 `Pattern.compile` |
| 泰 10.2 | 强制 | **避免用 Apache BeanUtils 做属性拷贝**（性能差）；可用 Spring BeanUtils、Cglib BeanCopier，注意都是浅拷贝 |
| 9.2 | 强制 | velocity 中直接用属性名取值；boolean 基本类型调用 isXxx()，Boolean 包装类型优先调用 getXxx() |
| 9.3 | 强制 | 后台传给页面的变量用 `$!{var}` |
| 9.4 | 强制 | `Math.random()` 的范围是 0 ≤ x < 1，可能取到 0，注意除零；要整数随机数就用 `Random.nextInt/nextLong` |
| 9.5 | 强制 | （泰山版移入（五）日期时间，见 ch11）获取毫秒数用 `System.currentTimeMillis()`；纳秒用 `nanoTime()`；JDK8 统计时间用 `Instant` |
| 9.6 | 强制 | （泰山版移入（五）日期时间，见 ch11）日期格式中的年份统一用小写 `y` |
| 9.7 | 推荐 | 视图模板中不写复杂逻辑 |
| 9.8 | 推荐 | 任何数据结构在构造或初始化时都要指定大小 |
| 9.9 | 推荐 | 及时清理无用的代码和配置；暂时注释掉、以后可能恢复的代码，用 `///` 说明原因 |

## Key Concepts
- **卫语句**：用提前 return 代替嵌套的 if-else。
- **等值判断被“击穿”**：并发下计数值越过 0 直接变成负数，`== 0` 永远不会成立。
- **week-year（YYYY）**：当天所在的周属于哪一年。
- **`$!{var}`**：velocity 中 var 为 null 时输出空字符串，而不是原样输出 `${var}`。

## Mental Models
- **先排除，再处理**：方法的前几行把所有不合格的情况 return 掉，剩下的就是主流程。
- **赋值点像人体的穴位**：赋值语句必须单独占一行，不要藏在条件或算术表达式里。
- **注释和代码的关系，就像导航软件和路网**：路网更新了导航没更新，导航就失去了意义。
- **用区间代替等值**：在并发环境中，`<= 0` 比 `== 0` 安全。

## Anti-patterns
- `switch (param)` 中 param 为 null：既不会进入 `case "null"`，也不会进入 default，而是直接抛 NPE。
- `if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))`：条件太复杂，难以理解。
- `return (sync = fair) ? new FairSync() : new NonfairSync();`：容易被误读成 `sync == fair`。
- 抽奖活动用“剩余数量 == 0”作为终止条件：并发下数量瞬间变成负数，活动无法终止。
- `new SimpleDateFormat("YYYY-MM-dd")`：每年最后一周显示成下一年。
- `// put elephant into fridge` 写在 `put(elephant, fridge);` 上方：多余的注释。

## Code Examples
```java
// 卫语句（原文示例）
public void findBoyfriend(Man man) {
    if (man.isUgly()) {
        System.out.println("本姑娘是外貌协会的资深会员");
        return;
    }
    if (man.isPoor()) {
        System.out.println("贫贱夫妻百事哀");
        return;
    }
    if (man.isBadTemper()) {
        System.out.println("银河有多远，你就给我滚多远");
        return;
    }
    System.out.println("可以先交往一段时间看看");
}

// 复杂条件赋值给布尔变量
final boolean existed = (file.open(fileName, "w") != null) && (...) || (...);
if (existed) {
    ...
}

// 日期格式
new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

// 泰山新增：三目运算符的拆箱 NPE
Integer a = 1, b = 2, c = null;
Boolean flag = false;
Integer result = (flag ? a * b : c);   // a*b 是 int，c 被拆箱成 int → NPE

// 正则预编译：定义成常量，而不是在方法体内编译
private static final Pattern MOBILE_PATTERN = Pattern.compile("^1\\d{10}$");
```
- **What it demonstrates**：用卫语句拆掉嵌套；用有意义的布尔变量提高可读性；日期格式的大小写。

## Worked Example
重构一段嵌套 4 层的发券逻辑：
```java
// 重构前
if (user != null) {
    if (user.isActive()) {
        if (stock.get() == 0) { stop(); } else {
            if (!(user.getLevel() >= 3)) { ... } else { send(user); }
        }
    }
}
```
按手册逐步处理：
1. 违反 7.5（嵌套超过 3 层）→ 改成卫语句：`if (user == null || !user.isActive()) { return; }`
2. 违反 7.4（并发下的等值判断）→ 改成 `if (stock.get() <= 0) { stop(); return; }`
3. 违反 7.9（取反逻辑）→ 改成 `if (user.getLevel() < 3) { ...; return; }`
4. 等级阈值 3 是魔法值 → 定义常量 `MIN_COUPON_LEVEL`（ch01）。
5. 主流程剩下一行 `send(user);`，结构一目了然。

## Key Takeaways
1. if-else 不超过 3 层，超过就用卫语句、策略模式或状态模式。
2. switch 必须有 default；String 类型的外部参数先判 null。
3. 所有 if/for/while 都要加大括号；高并发下用区间判断代替等值判断。
4. 类、属性、方法用 Javadoc；抽象方法写清职责；类上写作者和日期；枚举每一项都要注释。
5. 日期格式：`yyyy-MM-dd HH:mm:ss`，年份永远用小写 y。
6. 正则预编译成常量；获取时间用 `currentTimeMillis` 或 `Instant`；随机整数用 `Random.nextInt`。

## Connects To
- **ch04**：并发场景下的等值判断“击穿”问题。
- **ch07**：多层条件语句会让代码难以测试，卫语句也是提升可测性的手段。
- **ch01**：命名得当，就能少写注释。
- **ch11**：泰山版把日期格式、毫秒数等条款独立成（五）日期时间。
- **ch13 第 1 问**：三目运算符拆箱 NPE 的 JLS 依据与 JDK7/8 差异。
