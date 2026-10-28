# 第 11 章：编程规约 —（五）日期时间（泰山版新增）

## Core Idea
泰山版（v1.6.0）把散落在“其它”和“并发”中的日期条款合并，并新增闰年、闰月、`java.sql` 日期类等条款，单独成节。核心是：**日期不是普通整数**。格式符号区分大小写；一年不一定是 365 天；Calendar 的月份从 0 开始；`java.sql.*` 日期类型不能当通用时间类型用。

## Frameworks Introduced
- **日期格式四对字母**：`yyyy` 表示年（大写 `YYYY` 是 week-year，当周跨年时返回下一年）；`MM` 表示月，`mm` 表示分；`HH` 表示 24 小时制，`hh` 表示 12 小时制。
  - 标准写法：`yyyy-MM-dd HH:mm:ss`。
- **时间获取三档**：毫秒用 `System.currentTimeMillis()`，不用 `new Date().getTime()`；纳秒用 `System.nanoTime()`；JDK8 统计耗时等场景推荐用 `Instant`。
- **JDK8 替代表**（见 ch04 并发第 5 条）：`Instant` 替代 Date，`LocalDateTime` 替代 Calendar，`DateTimeFormatter` 替代 SimpleDateFormat（不可变、线程安全）。
- **闰年两问**：
  - 天数要用 API 算：`LocalDate.now().lengthOfYear()`、`LocalDate.of(2011, 1, 1).lengthOfYear()`，不要写死 365。
  - 2 月 29 日加一年后不可能还是 2 月 29 日，涉及“一年后的同一天”的逻辑要专门处理。

## 规约速查
> 编号为泰山版（五）日期时间中的编号。第 1、3 条对应华山版（九）其它的 9.6、9.5（见 ch05），泰山版移到本节；第 2 条的大小写说明在华山版整理中已作为 9.6 的补充出现，是否单列为新条款未能确认；第 4–7 条为泰山版新增（版本历史点名“日期时间的闰年、闰月问题”）。

| # | 级别 | 规则 |
|---|---|---|
| 1 | 强制 | 日期格式化的 pattern 中，年份统一用小写 `y` |
| 2 | 强制 | 分清大写 M（月）和小写 m（分），大写 H（24 小时制）和小写 h（12 小时制） |
| 3 | 强制 | 获取当前毫秒数用 `System.currentTimeMillis()`，不用 `new Date().getTime()` |
| 4 | 强制 | **程序中任何地方都不允许使用 `java.sql.Date`、`java.sql.Time`、`java.sql.Timestamp`** |
| 5 | 强制 | **不要在程序中写死一年为 365 天** |
| 6 | 推荐 | 避免公历闰年 2 月问题：2 月 29 日一年后的那一天不可能是 2 月 29 日 |
| 7 | 推荐 | 用枚举值指代月份（`Calendar.JANUARY` 等）；如果用数字，注意 Date、Calendar 的月份取值是 0–11 |

## Key Concepts
- **week-year（YYYY）**：当天所在的周属于哪一年；一周从周日开始、周六结束，只要本周跨年，就返回下一年。
- **java.sql.Date**：只记录日期、不记录时间，调用 `getHours()` 会抛异常。
- **java.sql.Time**：只记录时间、不记录日期，调用 `getYear()` 会抛异常。
- **java.sql.Timestamp**：构造时 `super((time/1000)*1000)`，秒和纳秒分别存在 fastTime 和 nanos 中；用 `java.util.Date.after(Date)` 与它比较时会触发 JDK BUG（JDK9 已修复），结果可能出乎意料。
- **0-based month**：JDK 原生注释写明 Month value is 0-based，0 表示一月。

## Mental Models
- **把日期当成带日历规则的值，而不是数字**：凡是 `+365`、`month = 2` 这类硬编码，都要换成日历 API 或枚举。
- **格式串大小写是语义，不是风格**：`YYYY`、`mm`、`hh` 写错不会报错，但会在跨年那一周、某个时刻悄悄出错。
- **持久层类型不出持久层**：`java.sql.*` 只是 JDBC 的适配类型，业务代码里一律用 `java.util.Date` 或 JDK8 的时间类。

## Anti-patterns
- `new SimpleDateFormat("YYYY-MM-dd")`：每年最后一周显示成下一年。
- `int[] dayArray = new int[365];`：闰年第 366 天数组越界。
- 一年期会员用 `calendar.add(Calendar.DATE, 365)` 计算到期日：跨过闰年 2 月时，到期日比“一年后的同一天”提前一天。
- `calendar.set(2020, 2, 1)` 以为是 2 月 1 日，实际是 3 月 1 日（月份从 0 开始）。
- 业务对象里用 `java.sql.Timestamp` 和 `java.util.Date` 混合比较。

## Code Examples
```java
// 正例：用 API 计算一年的天数
int daysOfThisYear = LocalDate.now().lengthOfYear();
int days2011 = LocalDate.of(2011, 1, 1).lengthOfYear();

// 反例：硬编码 365
Calendar calendar = Calendar.getInstance();
calendar.set(2020, 1, 26);             // 注意：月份参数 1 表示 2 月
calendar.add(Calendar.DATE, 365);      // 2020 年是闰年，结果比一年后的同一天早一天

// 正例：一年后的同一天交给日历 API（示意，原文未给出）
LocalDate expire = LocalDate.of(2020, 2, 29).plusYears(1);   // 2021-02-28

// 正例：月份用枚举
calendar.set(Calendar.MONTH, Calendar.FEBRUARY);
```
- **What it demonstrates**：一年的长度和月份编号都交给日历 API 或枚举处理，不要手工推算。

## Reference Tables
| 字母 | 含义 | 易混淆的写法 |
|---|---|---|
| `yyyy` | 当天所在的年 | `YYYY`：week-year |
| `MM` | 月 | `mm`：分钟 |
| `HH` | 24 小时制 | `hh`：12 小时制 |
| `ss` | 秒 | — |

| 需求 | 推荐写法 |
|---|---|
| 当前毫秒数 | `System.currentTimeMillis()` |
| 高精度耗时 | `System.nanoTime()`；JDK8 用 `Instant` |
| 线程安全的格式化 | JDK8 用 `DateTimeFormatter`；低版本用 ThreadLocal 包装的 SimpleDateFormat（ch04） |
| 一年多少天 | `LocalDate#lengthOfYear()` |
| 指代月份 | `Calendar.JANUARY` 等枚举，或 `java.time.Month` |

## Worked Example
Review 一段会员到期日的代码：
```java
private static final SimpleDateFormat SDF = new SimpleDateFormat("YYYY-MM-dd hh:mm");
public String expireDate(java.sql.Timestamp regTime) {
    Calendar c = Calendar.getInstance();
    c.setTime(regTime);
    c.add(Calendar.DATE, 365);
    return SDF.format(c.getTime());
}
```
逐条指出问题：
1. `YYYY` 违反第 1 条 → `yyyy`；`hh` 违反第 2 条，24 小时制要写 `HH`。
2. static SimpleDateFormat 违反 ch04 并发第 5 条 → 改用 `DateTimeFormatter`（原理见 ch13 第 9 问）。
3. 参数类型 `java.sql.Timestamp` 违反第 4 条 → 用 `LocalDateTime` 或 `java.util.Date`。
4. `+365` 违反第 5 条 → `regTime.plusYears(1)`；2 月 29 日注册的情况要单独确认业务规则（第 6 条）。
5. 重构后：`return regTime.plusYears(1).format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm"));`（示意）。

## Key Takeaways
1. 格式串固定写 `yyyy-MM-dd HH:mm:ss`，留意 Y/y、M/m、H/h 三对字母。
2. 毫秒用 `currentTimeMillis()`，纳秒用 `nanoTime()`，JDK8 统计时间用 `Instant`。
3. 业务代码禁用 `java.sql.Date/Time/Timestamp`。
4. 不写死 365 天，用 `lengthOfYear()`、`plusYears()`；注意 2 月 29 日。
5. 月份用枚举，Calendar 的月份从 0 开始。

## Connects To
- **ch04**：SimpleDateFormat 的线程安全问题与 DateTimeFormatter。
- **ch05**：华山版中 9.5、9.6 两条的旧位置。
- **ch13 第 9 问**：SimpleDateFormat 共享 calendar 成员变量导致的竞态。
