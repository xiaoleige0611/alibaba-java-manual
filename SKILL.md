---
name: alibaba-java-manual
description: "Knowledge base from \"《Java开发手册》v1.5.0 华山版\" by 阿里巴巴与 Java 社区开发者. Use when writing or reviewing Java code against 阿里巴巴 Java 开发规约 (命名/OOP/集合/并发/异常日志/单元测试/安全/MySQL 建表索引 SQL ORM/工程分层/二方库/设计规约), citing 【强制】【推荐】【参考】 rules, or studying the manual."
---

<!-- argument-hint: [topic, rule keyword, or chapter number] -->

# 《Java开发手册》v1.5.0 华山版
**作者**：阿里巴巴与 Java 社区开发者 | **页数**：约 44 | **章节**：10 | **生成日期**：2026-10-21 | **原版发布**：2019.06.19

## 安装

```bash
npx skills add https://github.com/xiaoleige0611/alibaba-java-manual --skill alibaba-java-manual
```

安装后重启 agent 会话。

## 如何使用
- **不带参数**：加载下面的核心规约，用于写代码或做 Code Review。
- **带主题**：例如 `线程池`、`BigDecimal`、`索引`、`日志`，我会定位到对应章节，读完再回答。
- **带章节**：例如 `ch04`，直接加载该章。
- **Code Review**：逐条对照规约指出违规之处，并注明级别（【强制】/【推荐】/【参考】）和章节号。

遇到下面核心部分没有覆盖的问题时，先读对应的章节文件再回答；需要引用条款时，以章节文件中的“规约速查”表为准。

---

## 核心规约与思维模型

**规约分三级**：【强制】违反就可能出故障，必须修改；【推荐】是最佳实践，偏离时要说明理由；【参考】是经验建议。每条规约的延伸信息分为“说明”（解释）、“正例”（推荐的写法）、“反例”（需要提防的雷区和真实故障）。

### 1. 命名与常量（ch01）
- 类名 UpperCamelCase（DO/BO/DTO/VO/AO/PO/UID 例外），方法和变量 lowerCamelCase，常量 `MAX_STOCK_COUNT`；禁止拼音、中文，禁止以 `_`/`$` 开头或结尾，禁止随意缩写。
- **POJO 布尔属性不加 is**（RPC 序列化会出错）；数据库字段用 `is_xxx`，两者靠 resultMap 映射。
- Service/DAO 方法前缀：get / list / count / save(insert) / remove(delete) / update；实现类加 `Impl`。
- **禁止魔法值**；long 字面量用大写 `L`；常量按功能分类，并按五层复用范围存放；取值范围固定时用 enum。

### 2. OOP 与格式（ch02）
- 4 个空格缩进，单行不超过 120 字符，方法不超过 80 行，文件 UTF-8，换行符用 Unix 格式；覆写方法加 `@Override`。
- 用常量调用 equals：`"test".equals(obj)`；**包装整型之间一律用 equals 比较**（IntegerCache 只覆盖 -128~127）。
- 浮点数不能用 `==` 或 equals 判断相等 → 用误差范围或 BigDecimal；**禁止 `new BigDecimal(double)`**。
- POJO 属性和 RPC 的参数、返回值用包装类型，局部变量用基本类型；POJO 不设默认值、必须写 toString、不能同时有 isXxx 和 getXxx。

### 3. 集合（ch03）
- 覆写 equals 就必须覆写 hashCode；Set 的元素和 Map 的 key 都要覆写这两个方法。
- subList、keySet、Arrays.asList、Collections.emptyList 是**视图或不可变对象**：不能强转，不能随意增删。
- 转数组用 `toArray(new T[0])`；**foreach 中不要 remove/add，用 Iterator**；Comparator 要处理相等的情况。
- 初始容量 = n / 0.75 + 1；遍历 Map 用 entrySet；ConcurrentHashMap 的 key 和 value 都不能为 null。

### 4. 并发（ch04）
- **线程池不用 Executors，用 ThreadPoolExecutor**（Fixed/Single 的队列无界，Cached 的线程数无界，都会 OOM）；线程要有有意义的名字。
- SimpleDateFormat 不要定义成 static（用 ThreadLocal，或 JDK8 的 DateTimeFormatter）；**ThreadLocal 必须在 finally 中 remove**。
- `lock()` 写在 try 外并紧挨着 try，在 finally 中 unlock；tryLock 要先判断结果；多资源加锁顺序一致。
- 锁粒度：能无锁就不加锁，能锁代码块就不锁方法，能用对象锁就不用类锁；锁内不调用 RPC。
- 冲突率低于 20% 用乐观锁（version，重试 ≥ 3 次），否则或涉及资金用悲观锁；定时任务用 ScheduledExecutorService，不用 Timer；DCL 的属性加 volatile；计数用 LongAdder。

### 5. 控制语句与注释（ch05）
- switch 必须有 default；String 类型的外部参数先判 null；if/for/while 都加大括号。
- **if-else 不超过 3 层 → 用卫语句、策略模式或状态模式**；复杂条件赋值给有意义的布尔变量；表达式中不插入赋值；避免取反逻辑。
- 高并发场景用区间判断代替等值判断。
- 类、属性、方法用 Javadoc；抽象方法写清做什么；类上写创建者和日期；枚举每一项都要注释；TODO/FIXME 注明标记人和时间。
- 日期格式用 `yyyy-MM-dd HH:mm:ss`（**大写 YYYY 是 week-year**）；正则预编译；获取毫秒数用 `currentTimeMillis()`。

### 6. 异常与日志（ch06）
- 能预检查的 RuntimeException 不要 catch；不用异常做流程控制；捕获了就要处理，否则往上抛。
- **finally 中不要 return**；关闭资源用 try-with-resources；调用 RPC 或二方包时 **catch Throwable**。
- 对外 http 接口用错误码，应用内部抛异常，**跨应用 RPC 用 Result**。防止 NPE 是调用者的责任（注意六大场景，可用 Optional）。
- 日志：**用 SLF4J**、`{}` 占位符、debug/info 先判断级别、`additivity=false`、异常日志带现场信息和 e；保留至少 15 天，敏感操作日志至少 6 个月；参数错误记 warn。

### 7. 单元测试与安全（ch07）
- **AIR**：自动化（用 assert，不用 System.out）、独立、可重复（依赖注入加 Mock）；**BCDE**：边界、正确、设计、错误。
- 语句覆盖率 70%，核心模块的语句和分支都要 100%；测试代码放在 `src/test/java`；数据库测试自动回滚。
- 安全：水平权限校验、敏感数据脱敏、SQL 参数绑定、所有参数做有效性校验、输出转义、CSRF 防护、防重放（限次数、限频率、验证码）。

### 8. MySQL（ch08）
- 表必备 `id bigint unsigned` 主键、`create_time`、`update_time`；表名和字段名用小写，表名不用复数；小数用 **decimal**；varchar ≤ 5000；索引命名 pk_/uk_/idx_。
- 业务唯一的字段**必须建唯一索引**；**join 不超过 3 个表**；禁止左模糊；varchar 索引要指定长度；组合索引等号列在前、区分度高的在前；explain 的 type 至少为 range。
- 用 `count(*)`；`IFNULL(SUM(col), 0)`；用 `ISNULL()` 判断 NULL；**不用外键、级联和存储过程**；订正数据前先 select；in 的元素不超过 1000 个。
- ORM：不用 `select *`；**用 `#{}`，不用 `${}`**；显式 resultMap；只更新有改动的字段；不滥用 `@Transactional`。

### 9. 工程结构（ch09）
- 分层：开放接口 / 终端显示 / Web / Service / **Manager**（封装第三方、下沉通用能力、组合多个 DAO）/ DAO。
- 异常按层处理：DAO 只抛不记；Service 记录日志并带参数；Web 跳转友好错误页；开放接口返回错误码。
- 领域模型：DO / DTO / BO / AO / VO / Query；**超过 2 个参数的查询禁止用 Map**。
- 二方库：GAV 规则，版本号格式 `主.次.修订` 从 1.0.0 开始，线上不依赖 SNAPSHOT，**接口返回值不能含枚举**，统一版本变量，升级时保持仲裁结果不变。
- 服务器：`tcp_fin_timeout=30`，调大 fd，加 `-XX:+HeapDumpOnOutOfMemoryError`，Xms 与 Xmx 相同。

### 10. 设计（ch10）
- 存储方案和数据结构（包括后续变更）必须评审通过并沉淀成文档。
- UML 阈值：用例超过 5 个画用例图；状态超过 3 个画状态图；调用链对象超过 3 个画时序图；模型类超过 5 个画类图；协作对象超过 2 个画活动图。
- 单一职责；组合优于继承（符合里氏代换）；依赖倒置；开闭原则；**设计的本质是找到变化点并隔离它**；同时评估异常流程。

---

## 章节索引

| # | 标题 | 关键内容 |
|---|---|---|
| [ch01](chapters/ch01-naming-constants.md) | 命名风格 + 常量定义 | 驼峰规则、布尔属性不加 is、方法前缀、魔法值、常量五层 |
| [ch02](chapters/ch02-format-oop.md) | 代码格式 + OOP 规约 | 120 字符 / 80 行、equals、IntegerCache、BigDecimal、包装类型、POJO |
| [ch03](chapters/ch03-collections.md) | 集合处理 | hashCode/equals、视图陷阱、toArray、PECS、Iterator 删除、Comparator、容量 |
| [ch04](chapters/ch04-concurrency.md) | 并发处理 | ThreadPoolExecutor、ThreadLocal、加锁模板、乐观锁与悲观锁、DCL、LongAdder |
| [ch05](chapters/ch05-control-comments-misc.md) | 控制语句 + 注释 + 其它 | switch、卫语句、参数校验取舍、Javadoc、TODO/FIXME、日期格式、正则 |
| [ch06](chapters/ch06-exception-logging.md) | 异常处理 + 日志规约 | 预检查、finally、Throwable、错误码 / 异常 / Result、NPE 六场景、SLF4J |
| [ch07](chapters/ch07-unit-test-security.md) | 单元测试 + 安全规约 | AIR、BCDE、覆盖率、越权、脱敏、SQL 注入、XSS、CSRF、防重放 |
| [ch08](chapters/ch08-mysql.md) | MySQL 数据库 | 建表三字段、decimal、唯一索引、join ≤ 3、组合索引、count/NULL、ORM `#{}` |
| [ch09](chapters/ch09-project-structure.md) | 工程结构 | 七层分层、Manager 层、分层异常、领域模型、GAV、版本号、服务器参数 |
| [ch10](chapters/ch10-design.md) | 设计规约 | 存储评审、UML 阈值、SOLID、隔离变化点、架构设计目的 |

## 主题索引

- **@Override / @Deprecated** → ch02
- **AIR / BCDE** → ch07
- **Arrays.asList / subList / toArray** → ch03
- **BigDecimal / 浮点比较** → ch02, ch08
- **CountDownLatch / Random / ThreadLocalRandom** → ch04
- **CSRF / XSS / SQL 注入 / 防重放** → ch07, ch08
- **DCL / volatile / LongAdder** → ch04
- **DO / DTO / BO / AO / VO / Query** → ch01, ch09
- **Executors / ThreadPoolExecutor / 线程命名** → ch04
- **explain / 覆盖索引 / 延迟关联 / 分页** → ch08
- **GAV / 二方库 / SNAPSHOT / 版本号** → ch09
- **hashCode / equals** → ch02, ch03
- **HashMap 容量 / null 值 / 死链** → ch03, ch04
- **if-else / 卫语句 / switch** → ch05
- **is_xxx / 布尔属性** → ch01, ch08
- **Javadoc / TODO / FIXME** → ch05
- **Lock / tryLock / 死锁 / 乐观锁 / 悲观锁** → ch04
- **Manager 层 / 分层** → ch09
- **NPE / Optional / 自动拆箱** → ch02, ch06
- **POJO / 包装类型 / toString / serialVersionUID** → ch02
- **Result / 错误码 / 自定义异常** → ch06, ch09
- **SimpleDateFormat / 日期格式 YYYY** → ch04, ch05
- **SLF4J / 日志级别 / 日志保留** → ch06
- **UML / 用例图 / 状态图 / 时序图 / 类图 / 活动图** → ch10
- **建表 / decimal / varchar / 分库分表** → ch08
- **开闭 / 依赖倒置 / 里氏代换 / 单一职责** → ch10
- **魔法值 / 常量 / enum** → ch01
- **索引 / 唯一索引 / 组合索引 / join** → ch08
- **服务器 / time_wait / fd / HeapDump / Xms** → ch09
- **参数校验** → ch05, ch07
- **单元测试 / 覆盖率 / Mock** → ch07

## 辅助文件

- [glossary.md](glossary.md) — 手册专有名词和关键术语
- [patterns.md](patterns.md) — 加锁模板、线程池、卫语句、Result、分层异常等可复用技法
- [cheatsheet.md](cheatsheet.md) — 决策表、阈值与默认值、问题信号

---

## 范围与限制

本 skill 只覆盖《Java开发手册》v1.5.0 华山版（2019.06.19）的内容。该版本基于 JDK8 时代编写；后续版本（如嵩山版、黄山版）新增或修改了部分条款，如果需要以最新版为准，请对照官方最新版本。章节内容是对规约的提炼与综合，不是原文；标注为“示意”的代码是根据规则编写的，原文中没有。
