# 第 1 章：编程规约 —（一）命名风格 +（二）常量定义

## Core Idea
命名的目标是**代码自解释**：一个名字要能读出它是什么（类型/角色）、做什么（语义）、属于哪一层（DO/DTO/Service）。常量的目标是**消灭魔法值**，并按复用范围分层存放。

## Frameworks Introduced
- **命名风格四件套**：UpperCamelCase（类）/ lowerCamelCase（方法、参数、成员变量、局部变量）/ CONSTANT_CASE（常量、枚举成员）/ 全小写单数（包名）。
  - When to use：任何新标识符。
  - How：类名例外允许 DO / BO / DTO / VO / AO / PO / UID 这类后缀保持大写；其余缩写一律按单词驼峰（`XmlService`，不是 `XMLService`）。
- **类型名词后置**：表示类型的名词放在词尾 —— `startTime` / `workQueue` / `nameList` / `TERMINATED_THREAD_COUNT`。
- **模式入名**：用了设计模式就在名字里体现，如 `OrderFactory`、`LoginProxy`、`ResourceObserver`。
- **接口与实现两套规则**：Service/DAO 暴露接口，实现类加 `Impl`（【强制】）；描述能力的接口用 -able 形容词（`Translatable`，【推荐】）。
- **Service/DAO 方法前缀约定**（【参考】）：`get` 单个 / `list` 多个（复数结尾，如 `listObjects`）/ `count` 统计 / `save|insert` 插入 / `remove|delete` 删除 / `update` 修改。
- **领域模型命名**：`xxxDO`（xxx = 表名）、`xxxDTO`（业务领域名）、`xxxVO`（一般是网页名）；POJO 是统称，**禁止**命名成 `xxxPOJO`。
- **常量复用五层**：跨应用（二方库 client.jar 的 constant 目录）→ 应用内（一方库子模块的 constant 目录）→ 子工程内 → 包内 → 类内（`private static final`）。

## 规约速查
> 泰山版本章（命名风格 18 条、常量定义 5 条）与华山版条数一致，未见新增条款。

| # | 级别 | 规则 |
|---|---|---|
| 1.1 | 强制 | 命名不以 `_` 或 `$` 开头或结尾 |
| 1.2 | 强制 | 禁止拼音与英文混用，禁止中文命名；`alibaba`/`hangzhou` 这类国际通用名视同英文 |
| 1.3 | 强制 | 类名 UpperCamelCase（DO/BO/DTO/VO/AO/PO/UID 例外） |
| 1.4 | 强制 | 方法、参数、成员变量、局部变量用 lowerCamelCase |
| 1.5 | 强制 | 常量全大写加下划线，语义完整，不嫌长（`MAX_STOCK_COUNT`，不写 `MAX_COUNT`） |
| 1.6 | 强制 | 抽象类以 `Abstract`/`Base` 开头；异常类以 `Exception` 结尾；测试类 = 被测类名 + `Test` |
| 1.7 | 强制 | 数组写 `int[] arr`，不写 `String args[]` |
| 1.8 | 强制 | **POJO 布尔属性不加 `is` 前缀**（否则 RPC 框架会误判属性名，引发序列化错误）。原理见 ch13 第 10 问 |
| 1.9 | 强制 | 包名全小写，每段只有一个英文单词，用单数；类名可以用复数（`MessageUtils`） |
| 1.10 | 强制 | 子类与父类成员变量不同名，不同代码块的局部变量不同名；非 setter 参数不与成员变量同名 |
| 1.11 | 强制 | 杜绝不规范缩写（`AbsClass`、`condi`） |
| 1.12–14 | 推荐 | 用完整单词组合；类型名词后置；模式入名 |
| 1.15 | 推荐 | 接口方法和属性不加任何修饰符（连 `public` 也不加），配 Javadoc；接口里尽量不定义变量 |
| 1.16 | 强制/推荐 | `XxxServiceImpl implements XxxService`；能力接口用 -able |
| 1.17 | 参考 | 枚举类加 `Enum` 后缀，成员全大写（`ProcessStatusEnum.UNKNOWN_REASON`） |
| 1.18 | 参考 | 各层命名规约：Service/DAO 方法前缀（见上）、领域模型命名（见上） |
| 2.1 | 强制 | **禁止魔法值**直接出现在代码中 |
| 2.2 | 强制 | long 字面量用大写 `L`（`2l` 看起来像 21） |
| 2.3 | 推荐 | 常量按功能分类（`CacheConsts`、`ConfigConsts`），不要一个大而全的常量类 |
| 2.4 | 推荐 | 常量按五层复用范围存放 |
| 2.5 | 推荐 | 取值只在固定范围内变化 → 用 enum；有附加属性时更应如此 |

## Key Concepts
- **魔法值**：未经预先定义就直接写在代码里的字面量。
- **UpperCamelCase / lowerCamelCase**：大驼峰 / 小驼峰。
- **POJO**：只有 setter/getter/toString 的简单类，DO/DTO/BO/VO 的统称。
- **一方库 / 二方库**：本工程子模块的 jar / 公司内部发布到仓库的 jar。

## Mental Models
- 把名字当成**微型文档**：读者不进方法体，单看名字就该知道它返回什么。
- 遇到布尔字段，想想**跨层映射**：数据库用 `is_deleted`，POJO 用 `deleted`，靠 `<resultMap>` 做映射（见 ch08）。
- 每写一个字面量就问一句：**这个值会不会在第二个地方再出现？** 会，就定义成常量，并放到最小但够用的那一层。

## Anti-patterns
- **`Boolean isDeleted` 属性**：生成的 getter 是 `isDeleted()`，RPC 框架反向解析时会以为属性叫 `deleted`，导致取不到值并抛异常。
- **两个类各自定义 `YES`**：`A.YES = "yes"`、`B.YES = "y"`，`A.YES.equals(B.YES)` 为 false，线上出故障 → 易懂的值也要统一成应用内共享常量。
- **魔法字符串拼缓存 key**：`"Id#taobao_" + tradeId` 复制时漏掉下划线，导致缓存击穿。
- **`TCPUDPDeal`、`UserDo`**：缩写全大写、DO 后缀写错大小写。

## Code Examples
```java
// 反例：同名遮蔽
public class ConfusingName {
    public int age;
    public void getData(String alibaba) {
        if (condition) {
            final int money = 531;
        }
        for (int i = 0; i < 10; i++) {
            final int money = 615;   // 与上面代码块中的 money 同名
        }
    }
}
class Son extends ConfusingName {
    public int age;                  // 与父类成员变量同名
}

// 正例：带附加属性的枚举
public enum SeasonEnum {
    SPRING(1), SUMMER(2), AUTUMN(3), WINTER(4);
    private int seq;
    SeasonEnum(int seq) { this.seq = seq; }
    public int getSeq() { return seq; }
}
```
- **What it demonstrates**：变量遮蔽会降低可读性；enum 可以承载名称之外的附加信息（季节序号）。

## Reference Tables
| 场景 | 正例 | 反例 |
|---|---|---|
| 类名 | `JavaServerlessPlatform` `UserDO` `XmlService` `TcpUdpDeal` `TaPromotion` | `javaserverlessplatform` `UserDo` `XMLService` `TCPUDPDeal` `TAPromotion` |
| 常量 | `MAX_STOCK_COUNT` `CACHE_EXPIRED_TIME` | `MAX_COUNT` `EXPIRED_TIME` |
| 类型后置 | `startTime` `nameList` | `startedAt` `listName` `QueueOfWork` |
| 接口方法 | `void commit();` | `public abstract void f();` |

## Worked Example
给“查询某个用户的订单列表”的接口命名，按手册逐步推导：
1. 暴露的是服务 → 定义接口 `OrderService`，实现类 `OrderServiceImpl`。
2. 返回多个对象 → 用 `list` 前缀加复数：`listOrders(Long userId)`。
3. 参数超过 2 个时（如再加状态和时间段）→ 封装成 `OrderQuery`，禁止用 Map 传参（见 ch09）。
4. 持久层对象 `OrderDO`（对应 `order` 表），对外传输用 `OrderDTO`，页面展示用 `OrderListVO`。
5. 订单状态是固定集合 → `OrderStatusEnum { WAIT_PAY, PAID, ... }`，每个成员都写注释（见 ch05 注释规约）。
6. 是否已删除 → DO 中写 `Boolean deleted`，表字段写 `is_deleted unsigned tinyint`，在 resultMap 里做映射。

## Key Takeaways
1. 名字要自解释：不缩写、不用拼音、类型名词放词尾。
2. POJO 布尔属性不加 `is`；数据库字段必须加 `is_`，两者靠 resultMap 映射。
3. 任何字面量都可能是魔法值，定义成常量后放到最小够用的复用层。
4. Service/DAO 方法用固定前缀（get/list/count/save/remove/update），调用方看前缀就能猜出语义。
5. long 字面量用大写 `L`；取值范围固定的值用 enum。

## Connects To
- **ch02**：POJO 规约（包装类型、toString、不能同时存在 isXxx 和 getXxx）。
- **ch08**：建表规约里的 `is_xxx` 字段与 resultMap 映射。
- **ch09**：分层领域模型 DO/DTO/BO/AO/VO/Query 的定义。
- **ch13 第 10 问**：`isSuccess` 命名在 fastjson、jackson、Gson 之间序列化结果不一致的根因。
