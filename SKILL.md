---
name: alibaba-java-manual
description: "Knowledge base from \"《Java开发手册》华山版 v1.5.0 + 泰山版 v1.6.0\" and \"《Java开发手册（泰山版）灵魂13问》\" by 阿里巴巴与 Java 社区开发者 / Hollis. Use when writing or reviewing Java code against 阿里巴巴 Java 开发规约 (命名/OOP/日期时间/集合/并发/错误码/异常日志/单元测试/安全/MySQL 建表索引 SQL ORM/工程分层/二方库/设计规约), citing 【强制】【推荐】【参考】 rules, explaining why a rule exists, or studying the manual."
---

<!-- argument-hint: [topic, rule keyword, or chapter number] -->

# 《Java开发手册》华山版 v1.5.0 + 泰山版 v1.6.0（含《灵魂13问》）
**作者**：阿里巴巴与 Java 社区开发者；《灵魂13问》作者 Hollis | **页数**：约 57 + 约 120 | **章节**：13 | **生成日期**：2026-10-28 | **原版发布**：华山版 2019.06.19，泰山版 2020.04.22

## 安装

```bash
npx skills add https://github.com/xiaoleige0611/alibaba-java-manual --skill alibaba-java-manual
```

安装后重启 agent 会话。

## 如何使用
- **不带参数**：加载下面的核心规约，用于写代码或做 Code Review。
- **带主题**：例如 `线程池`、`错误码`、`闰年`、`索引`，我会定位到对应章节，读完再回答。
- **带章节**：例如 `ch04`，直接加载该章。
- **问“为什么”**：例如“为什么不能用 Executors”，先读 ch13 对应的问题。
- **Code Review**：逐条对照规约指出违规之处，并注明级别（【强制】/【推荐】/【参考】）和章节号。

遇到下面核心部分没有覆盖的问题时，先读对应的章节文件再回答；需要引用条款时，以章节文件中的“规约速查”表为准（编号沿用华山版，`泰 x.y` 为泰山版新增条款的泰山版编号）。

---

## 泰山版（v1.6.0）带来的变化
- **新增两大节**：编程规约（五）**日期时间**（ch11）；异常日志（一）**错误码** + 附 3 错误码列表（ch12）。其余小节编号随之后移。
- **新增条款**：下文标“（泰山）”的都是；另有【推荐】中断逻辑后加空行、删除未使用的字段和参数（ch05），闰年 2 月、月份用枚举（ch11）。各章“规约速查”中以 `泰 x.y` 标出。
- **修订**：阻塞等待锁的描述（ch04 第 9 条）；ISNULL 示例（ch08）；设计规约调整为 19 条，新增“代码即文档的观点是错误的”（ch10）。
- **《灵魂13问》**：用 JLS、JDK 源码、反编译和实测数据解释 13 条规约的原理（ch13）。

---

## 核心规约与思维模型

**规约分三级**：【强制】违反就可能出故障，必须修改；【推荐】是最佳实践，偏离时要说明理由；【参考】是经验建议。每条规约的延伸信息分为“说明”“正例”“反例”。

### 1. 命名与常量（ch01）
- 类名 UpperCamelCase（DO/BO/DTO/VO/AO/PO/UID 例外），方法和变量 lowerCamelCase，常量 `MAX_STOCK_COUNT`；禁止拼音、中文，禁止以 `_`/`$` 开头或结尾，禁止随意缩写。
- **POJO 布尔属性不加 is**（序列化框架推导属性名不一致，ch13 第 10 问）；数据库字段用 `is_xxx`，两者靠 resultMap 映射。
- Service/DAO 方法前缀：get / list / count / save / remove / update；实现类加 `Impl`。**禁止魔法值**；long 用大写 `L`；固定取值范围用 enum。

### 2. OOP 与格式（ch02）
- 4 个空格缩进，单行 ≤ 120 字符，方法 ≤ 80 行，UTF-8 加 Unix 换行；覆写加 `@Override`。
- `"test".equals(obj)`；**包装整型之间一律用 equals**；浮点数不用 `==`/equals 判等；**禁止 `new BigDecimal(double)`**；（泰山）**货币金额用最小单位整型存储**。
- POJO 属性和 RPC 参数、返回值用包装类型；POJO 不设默认值、必须写 toString；不随意改 serialVersionUID（ch13 第 11 问）；循环内用 StringBuilder（ch13 第 6 问）。

### 3. 日期时间（ch11，泰山新增）
- 格式 `yyyy-MM-dd HH:mm:ss`（**YYYY 是 week-year**）；毫秒用 `currentTimeMillis()`；JDK8 用 Instant / LocalDateTime / DateTimeFormatter。
- **禁用 java.sql.Date/Time/Timestamp**；**不写死 365 天**（用 `lengthOfYear()`）；注意 2 月 29 日；月份用枚举（Calendar 月份从 0 开始）。

### 4. 集合（ch03）
- 覆写 equals 必须覆写 hashCode；subList、keySet、Arrays.asList、emptyList 是**视图或不可变对象**（ch13 第 5 问）。
- 转数组 `toArray(new T[0])`；**foreach 中不 remove/add，用 Iterator**（ch13 第 7 问）；Comparator 处理相等。
- （泰山）判空用 `isEmpty()`；**`Collectors.toMap` 必须传 mergeFunction，value 不能为 null**。
- 初始容量 = n / 0.75 + 1（ch13 第 2、3 问）；遍历 Map 用 entrySet；ConcurrentHashMap 的 K/V 不能为 null。

### 5. 并发（ch04）
- **线程池不用 Executors，用 ThreadPoolExecutor**（无界队列或无限线程 → OOM，ch13 第 4 问）；线程要命名。
- SimpleDateFormat 不定义成 static（ch13 第 9 问）；**ThreadLocal 在 finally 中 remove**。
- `lock()` 写在 try 外并紧挨 try，二者之间不能有可能抛异常的调用，finally 中 unlock；tryLock 先判断结果；多资源加锁顺序一致。
- 锁粒度尽量小；锁内不调 RPC；冲突率 < 20% 用乐观锁（重试 ≥ 3 次），资金用悲观锁；用 ScheduledExecutorService 不用 Timer；DCL 加 volatile；计数用 LongAdder。

### 6. 控制语句与注释（ch05）
- switch 必须有 default；String 外部参数先判 null；if/for/while 加大括号；高并发用区间判断代替等值判断。
- （泰山）**三目运算符类型对齐会自动拆箱 → NPE**（ch13 第 1 问）；**避免 Apache BeanUtils 拷贝属性**。
- **if-else 不超过 3 层 → 卫语句、策略模式、状态模式**；复杂条件赋值给布尔变量；表达式中不插入赋值。
- 类、属性、方法用 Javadoc；类上写作者和日期；枚举每项注释；TODO/FIXME 注明标记人和时间；正则预编译。

### 7. 错误码（ch12，泰山新增）
- 5 位字符串：**A 用户 / B 本系统 / C 第三方** + 四位编号，大类步长 100；正常时 `00000`。
- 不体现版本号和等级；不挂钩组织架构；申请后永久固定；优先复用附表；**不直接给用户看**；第三方码可转义为 B 并保留原码；与 HTTP 状态码无关。

### 8. 异常与日志（ch06）
- 能预检查的 RuntimeException 不 catch；不用异常做流程控制；捕获就要处理，否则往上抛。
- **finally 中不 return**；try-with-resources；调 RPC 或二方包时 **catch Throwable**。
- 对外 http 接口用错误码，应用内部抛异常，**跨应用 RPC 用 Result**；防 NPE 是调用者的责任。
- 日志：**只用 SLF4J/JCL 门面**（ch13 第 8 问）、`{}` 占位符、debug/info 先判断级别、`additivity=false`、带现场信息和 e；至少保留 15 天；参数错误记 warn；（泰山）生产环境禁用 System.out 和 printStackTrace，不用 JSON 工具序列化对象打日志。

### 9. 单元测试与安全（ch07）
- **AIR**：自动化（用 assert）、独立、可重复；**BCDE**：边界、正确、设计、错误；语句覆盖率 70%，核心模块 100%。
- 安全：水平权限、脱敏、SQL 参数绑定、参数校验、输出转义、CSRF、（泰山）**外部重定向白名单**、防重放。

### 10. MySQL（ch08）
- 表必备 `id bigint unsigned`、`gmt_create`、`gmt_modified`（泰山版原文）；小写、不用复数；小数用 **decimal**；varchar ≤ 5000；索引 pk_/uk_/idx_。
- 业务唯一字段**建唯一索引**；**join ≤ 3 表**；禁止左模糊；组合索引等号列在前；explain 至少 range。
- 用 `count(*)`（ch13 第 13 问）；`IFNULL(SUM(col), 0)`；`ISNULL()` 判 NULL；不用外键、级联、存储过程；（泰山）**多表 SQL 列名加别名**，别名 `as t1`。
- ORM：不用 `select *`；**用 `#{}` 不用 `${}`**；显式 resultMap；只更新改动字段。

### 11. 工程结构（ch09）
- 分层：开放接口 / 终端显示 / Web / Service / **Manager** / DAO；DAO 只抛不记，Service 记日志，Web 跳友好页，开放接口转错误码。
- DO / DTO / BO / AO / VO / Query；**超过 2 个参数的查询禁止用 Map**。
- 二方库：GAV、`主.次.修订` 从 1.0.0 起、线上不依赖 SNAPSHOT、**返回值不含枚举**、统一版本变量；（泰山）不用不稳定的工具包。
- 服务器：`tcp_fin_timeout=30`、调大 fd、HeapDump、Xms = Xmx。

### 12. 设计（ch10）
- 存储方案和数据结构变更必须评审；UML 阈值：用例 > 5、状态 > 3、调用链对象 > 3、模型类 > 5、协作对象 > 2。
- 单一职责；**组合优于继承**（ch13 第 12 问）；依赖倒置；开闭原则；可扩展性的本质是隔离变化点；“代码即文档”是错误观点。

---

## 章节索引

| # | 标题 | 关键内容 |
|---|---|---|
| [ch01](chapters/ch01-naming-constants.md) | 命名风格 + 常量定义 | 驼峰、布尔属性不加 is、方法前缀、魔法值、常量五层 |
| [ch02](chapters/ch02-format-oop.md) | 代码格式 + OOP 规约 | 120 字符 / 80 行、equals、IntegerCache、BigDecimal、货币整型、POJO |
| [ch03](chapters/ch03-collections.md) | 集合处理 | hashCode/equals、视图陷阱、isEmpty、toMap、toArray、Iterator、容量 |
| [ch04](chapters/ch04-concurrency.md) | 并发处理 | ThreadPoolExecutor、ThreadLocal、加锁模板、乐观锁与悲观锁、DCL |
| [ch05](chapters/ch05-control-comments-misc.md) | 控制语句 + 注释 + 其它 | switch、三目拆箱、卫语句、Javadoc、BeanUtils、正则 |
| [ch06](chapters/ch06-exception-logging.md) | 异常处理 + 日志规约 | 预检查、finally、Throwable、Result、NPE、SLF4J、禁用 printStackTrace |
| [ch07](chapters/ch07-unit-test-security.md) | 单元测试 + 安全规约 | AIR、BCDE、覆盖率、越权、脱敏、XSS、CSRF、重定向白名单、防重放 |
| [ch08](chapters/ch08-mysql.md) | MySQL 数据库 | 建表三字段、decimal、唯一索引、组合索引、count/NULL、表别名、`#{}` |
| [ch09](chapters/ch09-project-structure.md) | 工程结构 | 分层、Manager 层、分层异常、领域模型、GAV、版本号、服务器参数 |
| [ch10](chapters/ch10-design.md) | 设计规约 | 存储评审、UML 阈值、SOLID、隔离变化点、代码即文档的误区 |
| [ch11](chapters/ch11-datetime.md) | 日期时间（泰山新增） | yyyy/YYYY、M/m、H/h、java.sql 日期类、365 天、闰年、月份枚举 |
| [ch12](chapters/ch12-error-codes.md) | 错误码（泰山新增） | A/B/C 来源、5 位结构、00000、宏观错误码、转义、附 3 码表 |
| [ch13](chapters/ch13-soul-13-questions.md) | 《灵魂13问》 | 三目 NPE、HashMap 容量、Executors、subList、`+` 拼接、foreach 删除、日志门面、SimpleDateFormat、isSuccess、serialVersionUID、继承、count(*) |

## 主题索引

- **@Override / @Deprecated** → ch02
- **AIR / BCDE** → ch07
- **Apache BeanUtils / 属性拷贝** → ch05
- **Arrays.asList / subList / toArray** → ch03, ch13
- **BigDecimal / 浮点比较 / 货币金额** → ch02, ch08
- **Collectors.toMap / mergeFunction / isEmpty** → ch03
- **count(*) / count(1) / count(列)** → ch08, ch13
- **CountDownLatch / Random / ThreadLocalRandom** → ch04
- **CSRF / XSS / SQL 注入 / 防重放 / 重定向白名单** → ch07, ch08
- **DCL / volatile / LongAdder** → ch04
- **DO / DTO / BO / AO / VO / Query** → ch01, ch09
- **Executors / ThreadPoolExecutor / 线程命名** → ch04, ch13
- **explain / 覆盖索引 / 延迟关联 / 分页** → ch08
- **foreach 删除 / Iterator / modCount / fail-fast** → ch03, ch13
- **GAV / 二方库 / SNAPSHOT / 版本号** → ch09
- **hashCode / equals** → ch02, ch03
- **HashMap 容量 / 负载因子 / null 值 / 死链** → ch03, ch04, ch13
- **if-else / 卫语句 / switch / 三目运算符** → ch05, ch13
- **is_xxx / 布尔属性 / isSuccess** → ch01, ch08, ch13
- **Javadoc / TODO / FIXME** → ch05
- **java.sql.Date / 闰年 / 365 天 / 月份** → ch11
- **Lock / tryLock / 死锁 / 乐观锁 / 悲观锁** → ch04
- **Manager 层 / 分层** → ch09
- **NPE / Optional / 自动拆箱** → ch02, ch05, ch06, ch13
- **POJO / 包装类型 / toString / serialVersionUID** → ch02, ch13
- **Result / 错误码 / 自定义异常** → ch06, ch09, ch12
- **SimpleDateFormat / DateTimeFormatter / 日期格式 YYYY** → ch04, ch05, ch11, ch13
- **SLF4J / 日志门面 / 日志级别 / 日志保留 / printStackTrace** → ch06, ch13
- **StringBuilder / 字符串拼接** → ch02, ch13
- **UML / 用例图 / 状态图 / 时序图 / 类图 / 活动图** → ch10
- **建表 / decimal / varchar / gmt_create / 分库分表** → ch08
- **开闭 / 依赖倒置 / 里氏代换 / 单一职责 / 继承与组合** → ch10, ch13
- **魔法值 / 常量 / enum** → ch01
- **索引 / 唯一索引 / 组合索引 / join / 表别名** → ch08
- **服务器 / time_wait / fd / HeapDump / Xms** → ch09
- **参数校验** → ch05, ch07
- **单元测试 / 覆盖率 / Mock** → ch07
- **错误码 / A0001 / B0001 / C0001 / 00000** → ch12

## 辅助文件

- [glossary.md](glossary.md) — 手册专有名词和关键术语
- [patterns.md](patterns.md) — 加锁模板、线程池、卫语句、Result、错误码、toMap、日期计算等可复用技法
- [cheatsheet.md](cheatsheet.md) — 决策表、阈值与默认值、问题信号

---

## 范围与限制

本 skill 覆盖《Java开发手册》华山版 v1.5.0（2019.06.19）和泰山版 v1.6.0（2020.04.22），以及《Java开发手册（泰山版）灵魂13问》。华山版与泰山版的差异是根据条款数量和编号对照现有华山版整理得出的，只列出能确认的新增与修订。泰山版原文中表必备字段为 `gmt_create/gmt_modified`，日志保留只要求至少 15 天（未见“敏感操作日志 6 个月”）；此前 skill 中与之不符的写法已按泰山版原文改正。后续嵩山版、黄山版还有修改，需要以最新版为准时请对照官方版本。章节内容是对规约的提炼与综合，不是原文；标注为“示意”的代码是根据规则编写的，原文中没有。《灵魂13问》第 12 问在提取文本中不完整，只整理了已有内容。
