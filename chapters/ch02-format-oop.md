# 第 2 章：编程规约 —（三）代码格式 +（四）OOP 规约

## Core Idea
格式规约让所有人写出来的代码“长得一样”，降低阅读成本；OOP 规约专门堵住 Java 语言里最常见的几个坑：`equals` 空指针、Integer 缓存、浮点比较、BigDecimal 精度、自动拆箱 NPE、POJO 默认值。

## Frameworks Introduced
- **大括号四规则**：左大括号前不换行、左大括号后换行、右大括号前换行、右大括号后如果还有 `else` 等代码则不换行；空块直接写成 `{}`。
- **空格规则**：保留字（if/for/while/switch/do）与括号之间加空格；小括号内侧不加空格；二目、三目运算符两侧各加一个空格；`//` 后加一个空格；逗号后加空格；强制类型转换的右括号后不加空格（`(int)first`）。
- **120 字符换行五原则**：第二行相对第一行缩进 4 格，从第三行起不再继续缩进；运算符跟着下文一起换行；方法调用的 `.` 跟着下文一起换行；多个参数在逗号后换行；不要在括号前换行。
- **基本类型与包装类型选用标准**：POJO 属性用包装类型（【强制】）；RPC 方法的参数和返回值用包装类型（【强制】）；局部变量用基本类型（【推荐】）。
  - Why：`null` 能表达“远程调用失败”这类附加信息；基本类型的默认值 0 会把错误伪装成正常数据。
- **访问控制从严八条**：不允许外部直接 new 的类，构造方法 private；工具类不能有 public/default 构造方法；与子类共享的非 static 成员变量用 protected；仅本类使用的用 private；static 成员变量仅本类使用时用 private，并考虑是否加 final；仅类内部调用的方法用 private；仅对子类公开的方法用 protected。
- **方法排序**：public/protected 方法 > private 方法 > getter/setter；重载的方法和多个构造方法放在一起（此条优先）。

## 规约速查
> 编号沿用华山版；标 `泰 x.y` 的行是泰山版（v1.6.0）新增条款，x.y 为泰山版编号（泰山版插入新条款后，后续编号整体后移）。

| # | 级别 | 规则 |
|---|---|---|
| 3.5 | 强制 | 4 个空格缩进，禁用 tab（IDEA 不勾选 Use tab character；Eclipse 勾选 insert spaces for tabs） |
| 3.8 | 强制 | 单行不超过 120 个字符 |
| 3.10 | 强制 | 文件编码 UTF-8，换行符用 Unix 格式 |
| 3.11 | 推荐 | 单个方法不超过 **80 行**（包括签名、大括号、空行） |
| 3.12 | 推荐 | 不要为了对齐等号额外加空格 |
| 3.13 | 推荐 | 不同逻辑之间空**一**行，没必要空多行 |
| 4.1 | 强制 | 静态成员用类名访问，不要通过对象引用访问 |
| 4.2 | 强制 | 覆写方法必须加 `@Override` |
| 4.3 | 强制 | 只有参数类型和业务含义都相同时才用可变参数，不用 Object；可变参数放在参数列表最后 |
| 4.4 | 强制 | 外部在调用或二方库依赖的接口，不准改方法签名；接口过时要加 `@Deprecated` 并写明替代方案 |
| 4.5 | 强制 | 不使用过时的类或方法 |
| 4.6 | 强制 | 用常量或确定有值的对象调用 equals：`"test".equals(obj)`，推荐 `Objects.equals` |
| 4.7 | 强制 | **整型包装类之间的值比较一律用 equals** |
| 泰 4.8 | 强制 | **任何货币金额都以最小货币单位、整型类型存储**（如以“分”为单位存 long） |
| 4.8 | 强制 | 浮点数等值判断：基本类型不用 `==`，包装类型不用 `equals` |
| 4.9 | 强制 | DO 属性类型与数据库字段类型匹配（bigint ↔ Long） |
| 4.10 | 强制 | **禁止 `new BigDecimal(double)`** |
| 4.11 | 强制/推荐 | 包装类型与基本类型的选用标准（见上） |
| 4.12 | 强制 | POJO 不设属性默认值 |
| 4.13 | 强制 | 序列化类新增属性时不改 serialVersionUID；完全不兼容的升级才修改。原理见 ch13 第 11 问 |
| 4.14 | 强制 | 构造方法里不写业务逻辑，初始化逻辑放到 init 方法 |
| 4.15 | 强制 | POJO 必须写 toString；继承另一个 POJO 时调用 `super.toString` |
| 4.16 | 强制 | 同一属性不能同时有 `isXxx()` 和 `getXxx()` |
| 4.17 | 推荐 | 按索引访问 split 结果前，先检查最后一个分隔符后面有没有内容 |
| 4.20 | 推荐 | getter/setter 里不写业务逻辑 |
| 4.21 | 推荐 | 循环体内拼接字符串用 `StringBuilder.append`。原理见 ch13 第 6 问 |
| 4.22 | 推荐 | 以下情况用 final：不允许继承的类、不允许修改引用的域对象、不允许覆写的方法、不允许重新赋值的局部变量 |
| 4.23 | 推荐 | 慎用 clone（默认是浅拷贝） |

## Key Concepts
- **IntegerCache**：-128 到 127 之间的 Integer 会复用缓存对象，用 `==` 比较碰巧成立；超出这个范围就不成立。
- **尾数 + 阶码**：浮点数的二进制编码方式，绝大部分十进制小数无法精确表示。
- **serialVersionUID**：序列化版本号，前后不一致会抛序列化运行时异常。
- **浅拷贝**：`Object.clone` 默认只复制引用。

## Mental Models
- **包装类型的 null 是一种信号**：“没拿到值”和“值为 0”是两件事，页面应该显示 `-`，而不是 `0%`。
- **把 `==` 当成比较身份，`equals` 当成比较值**：对 Integer 用 `==` 是在赌缓存，迟早会出错。
- **需要精确计算就用 BigDecimal**，而且从 String 或 `valueOf` 构造。
- **给方法分出红花和绿叶**：方法超过 80 行时，把“绿叶”（辅助逻辑）抽成独立方法，把共性逻辑抽成公共方法。

## Anti-patterns
- `object.equals("test")`：object 为 null 时抛 NPE。
- `Integer a = 1000, b = 1000; a == b` 结果为 false。
- `new BigDecimal(0.1f)` 实际存储的是 0.10000000149。
- 表 id 是 `bigint unsigned`，DO 属性却用 `Integer`：id 增长到一定程度溢出变成负数。
- POJO 里 `createTime = new Date()`：更新其他字段时顺带把创建时间改成了当前时间。
- `"a,b,c,,".split(",").length` 是 3，不是 5。
- 在 getter 里写 `if (condition) return data + 100;`：排查问题时很难发现。

## Code Examples
```java
// 浮点等值判断：反例
float a = 1.0f - 0.9f;
float b = 0.9f - 0.8f;
if (a == b) { }                       // false
if (Float.valueOf(a).equals(Float.valueOf(b))) { }   // false

// 正例 1：设定误差范围
float diff = 1e-6f;
if (Math.abs(a - b) < diff) { }

// 正例 2：用 BigDecimal 计算
BigDecimal x = new BigDecimal("1.0").subtract(new BigDecimal("0.9"));
BigDecimal y = new BigDecimal("0.9").subtract(new BigDecimal("0.8"));
if (x.equals(y)) { }                  // true

// BigDecimal 的构造方式
BigDecimal recommend1 = new BigDecimal("0.1");
BigDecimal recommend2 = BigDecimal.valueOf(0.1);   // 内部调用 Double.toString
```
- **What it demonstrates**：浮点数不能精确比较；BigDecimal 必须从 String 或 `valueOf` 构造。

```java
// 120 字符换行：点号跟着方法名一起换行，第二行缩进 4 格，之后不再继续缩进
sb.append("Jack").append("Ma")...
    .append("alibaba")...
    .append("alibaba");
// 反例：在括号前换行；在逗号前换行
sb.append("Jack")...append
    ("alibaba");
method(args1, args2, args3, ...
    , argsX);
```

## Worked Example
对一个 DTO 做 Code Review，原始代码：
```java
public class TradeDTO {
    private boolean isSuccess;          // ①
    private int rate;                   // ②
    private Date createTime = new Date();   // ③
    private BigDecimal amount = new BigDecimal(0.1);   // ④
}
```
按手册逐条指出问题：
1. ① 违反 1.8 和 4.11：布尔属性不加 `is`，POJO 用包装类型 → `private Boolean success;`
2. ② 违反 4.11：RPC 调用失败时返回默认值 0，页面显示 0%，应该显示 `-` → `private Integer rate;`
3. ③ 违反 4.12：POJO 不设默认值，否则更新时会覆盖创建时间。
4. ④ 违反 4.10 和 4.12 → 去掉默认值，需要赋值时用 `BigDecimal.valueOf(...)` 或 `new BigDecimal("0.1")`。
5. 缺少 `toString()`（4.15）；如果实现了 Serializable，新增字段时不要改 serialVersionUID（4.13）。

## Key Takeaways
1. 格式：4 空格缩进、单行不超过 120 字符、UTF-8 加 Unix 换行、方法不超过 80 行。
2. 比较：equals 由常量一方调用；包装整型用 equals；浮点数用误差范围或 BigDecimal。
3. BigDecimal 禁用 double 构造方法。
4. POJO：属性用包装类型、不设默认值、必须写 toString、不能同时有 isXxx 和 getXxx。
5. 访问权限尽量收窄：能 private 就不 protected，能 protected 就不 public。

## Connects To
- **ch01**：布尔属性不加 is 前缀。
- **ch06**：自动拆箱是 NPE 的主要来源之一。
- **ch08**：建表时小数一律用 decimal，与 BigDecimal 呼应。
- **ch13**：第 6 问（循环里 `+` 拼接的字节码真相）、第 11 问（serialVersionUID 与 InvalidClassException）。
