# 第 8 章：MySQL 数据库 — 建表 / 索引 / SQL 语句 / ORM 映射

## Core Idea
表结构是最难修改的代码：字段名无法预发布，数据迁移代价很高，所以建表要**一次做对**（命名、类型、必备字段）；索引要基于**最左前缀和区分度**来设计；SQL 要避开 NULL、count、外键、存储过程的陷阱；ORM 层要**显式映射、不用 `*`、不用 `${}`**。

## Frameworks Introduced
- **表必备三字段**：`id`（bigint unsigned，主键，单表自增，步长为 1）、`create_time`、`update_time`（都是 datetime 类型）。
- **索引命名**：`pk_字段名`（主键）、`uk_字段名`（唯一索引）、`idx_字段名`（普通索引）。
- **is_xxx 布尔字段**：表示是与否的字段用 `is_xxx` 命名，类型是 `unsigned tinyint`（1 表示是，0 表示否）；POJO 属性不加 is，在 resultMap 中做映射。任何非负数的字段都必须是 unsigned。
- **冗余字段三条件**：不是频繁修改的字段；不是超长 varchar，更不能是 text；不是唯一索引的字段。
- **分库分表阈值**：单表行数超过 **500 万** 或单表容量超过 **2GB** 才考虑；三年内达不到这个量级就不要分。
- **explain type 目标**：至少达到 `range`，要求是 `ref`，最好是 `consts`；`index`（索引文件全扫描）很慢。
- **组合索引顺序**：区分度最高的放最左边；等号条件和非等号条件混合时，**等号条件的列放前面**（`where c>? and d=?` → `idx_d_c`）。
- **order by 利用索引**：`where a=? and b=? order by c` → 索引 `a_b_c`；索引中存在范围查询时无法利用有序性（`a>10 order by b` 用不上 `a_b` 排序）。
- **索引的三种极端误解**：宁滥勿缺（一个查询建一个索引）；宁缺勿滥（认为索引拖慢写入）；抵制唯一索引（坚持在应用层先查后插）。

## 规约速查
| # | 级别 | 规则 |
|---|---|---|
| 建 1 | 强制 | 是与否用 `is_xxx unsigned tinyint`；非负数一律 unsigned |
| 建 2 | 强制 | 表名、字段名用小写字母或数字；不能以数字开头；两个下划线之间不能只有数字（`level3_name` ✓ `level_3_name` ✗） |
| 建 3 | 强制 | 表名不用复数 |
| 建 4 | 强制 | 不用保留字（desc、range、match、delayed） |
| 建 5 | 强制 | 索引命名 pk_ / uk_ / idx_ |
| 建 6 | 强制 | 小数用 **decimal**，禁用 float/double；超出 decimal 范围就拆成整数和小数分开存 |
| 建 7 | 强制 | 长度几乎相等的字符串用 char |
| 建 8 | 强制 | varchar 不超过 **5000**；超过就用 text 并拆到独立的表 |
| 建 9 | 强制 | 表必备三字段 |
| 建 10–12 | 推荐 | 表名用“业务名称_表的作用”（`alipay_task`）；库名与应用名一致；修改字段含义时更新注释 |
| 建 13–14 | 推荐 | 适当冗余；500 万行或 2GB 再分库分表 |
| 索 1 | 强制 | 业务上唯一的字段（包括组合字段）必须建唯一索引（墨菲定律：没有唯一索引就一定会有脏数据） |
| 索 2 | 强制 | **超过 3 个表禁止 join**；join 的字段类型必须一致，被关联的字段要有索引 |
| 索 3 | 强制 | varchar 上建索引必须指定长度，按区分度确定 |
| 索 4 | 强制 | 页面搜索禁止左模糊和全模糊，需要时走搜索引擎 |
| 索 5–10 | 推荐 | order by 利用索引；覆盖索引（extra 显示 using index）；用延迟关联优化深分页；type 至少 range；区分度高的放左边；防止隐式类型转换导致索引失效 |
| SQL 1 | 强制 | 用 `count(*)`，不用 count(列名) 或 count(常量) 代替 |
| SQL 2 | 强制 | `count(distinct col1, col2)` 中只要有一列全为 NULL，即使另一列有不同的值也返回 0 |
| SQL 3 | 强制 | 某列全为 NULL 时 sum 返回 NULL → 写成 `IFNULL(SUM(col), 0)` |
| SQL 4 | 强制 | 用 `ISNULL()` 判断 NULL；`NULL=NULL`、`NULL<>NULL` 的结果都是 NULL |
| SQL 5 | 强制 | 分页查询时 count 为 0 就直接返回 |
| SQL 6 | 强制 | **不使用外键和级联**，外键概念在应用层解决 |
| SQL 7 | 强制 | **禁止使用存储过程** |
| SQL 8 | 强制 | 数据订正（删除、修改）前先 select 确认 |
| SQL 9 | 推荐 | 尽量避免 in；实在避免不了，in 后面的元素控制在 **1000** 个以内 |
| SQL 10–11 | 参考 | 用 utf-8 编码，需要存表情时用 utf8mb4；不建议在开发代码中使用 TRUNCATE（无事务、不触发 trigger） |
| ORM 1 | 强制 | 不用 `*` 作为查询字段列表 |
| ORM 2 | 强制 | POJO 布尔属性不加 is，数据库字段加 is_，在 resultMap 中映射 |
| ORM 3 | 强制 | 不用 resultClass 作为返回参数；每个表都有对应的 POJO |
| ORM 4 | 强制 | 用 `#{}`、`#param#`，**不用 `${}`**（SQL 注入） |
| ORM 5 | 强制 | 不用 iBATIS 的 `queryForList(statementName, start, size)`（它会取出全部记录再 subList） |
| ORM 6 | 强制 | 不直接用 HashMap 或 Hashtable 作为查询结果集的输出 |
| ORM 7 | 强制 | 更新记录时同时更新 `gmt_modified` |
| ORM 8 | 推荐 | 不写大而全的更新接口，只更新有改动的字段 |
| ORM 9 | 参考 | 不滥用 `@Transactional`；回滚方案要考虑缓存、搜索引擎、消息、统计 |

## Key Concepts
- **区分度**：`count(distinct left(列名, 索引长度)) / count(*)`；长度为 20 的索引，区分度一般能达到 90% 以上。
- **最左前缀匹配**：B-Tree 索引左边的值不确定时（左模糊）无法使用索引。
- **覆盖索引**：查询需要的列都在索引里，不需要回表。
- **延迟关联**：先用子查询定位 id，再 join 回原表取数据。
- **file_sort**：无法利用索引的有序性时发生的额外排序。

## Mental Models
- **索引就像书的目录**：想知道第 11 章的标题，看目录就够了，不用翻到那一页（覆盖索引）。
- **把唯一性交给数据库**：应用层的校验再完善，并发下也会漏，唯一索引是最后一道闸门。
- **NULL 不是值，而是“未知”**：和 NULL 做任何比较都得到 NULL，count、sum 的结果都受它影响。
- **数据库只做存储和检索**：外键、级联、存储过程把业务逻辑放进了数据库，在分布式和高并发场景下会变成瓶颈。

## Anti-patterns
- 表 id 用 `bigint unsigned`，DO 属性却是 Integer（溢出）。
- `LIMIT 100000, 20`：MySQL 会取出 100020 行再丢弃前 100000 行。
- `where name like '%张'`：索引失效。
- 存金额用 double：比较时得到错误结果。
- `select *`：增加解析成本，与 resultMap 不一致，text 字段浪费网络。
- `update table set c1=?, c2=?, c3=?` 更新所有字段：容易出错、效率低、增加 binlog。

## Code Examples
```sql
-- 建表模板（根据规则组合的示意）
CREATE TABLE trade_config (
    id           BIGINT UNSIGNED  NOT NULL AUTO_INCREMENT,
    config_name  VARCHAR(64)      NOT NULL COMMENT '配置名',
    amount       DECIMAL(18, 2)   NOT NULL COMMENT '金额',
    is_deleted   TINYINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '1 删除 0 未删除',
    create_time  DATETIME         NOT NULL,
    update_time  DATETIME         NOT NULL,
    PRIMARY KEY (id),
    UNIQUE KEY uk_config_name (config_name)
);

-- 延迟关联优化深分页（原文正例）
SELECT a.* FROM 表1 a, (SELECT id FROM 表1 WHERE 条件 LIMIT 100000, 20) b WHERE a.id = b.id;

-- sum 的 NPE 防护
SELECT IFNULL(SUM(column), 0) FROM table;

-- 字符统计函数的区别
SELECT LENGTH("轻松工作");            -- 12
SELECT CHARACTER_LENGTH("轻松工作");  -- 4
```

## Reference Tables
| 对象 | 年龄区间 | 类型 | 字节 | 无符号范围 |
|---|---|---|---|---|
| 人 | 150 岁之内 | tinyint unsigned | 1 | 0 到 255 |
| 龟 | 数百岁 | smallint unsigned | 2 | 0 到 65535 |
| 恐龙化石 | 数千万年 | int unsigned | 4 | 0 到约 42.9 亿 |
| 太阳 | 约 50 亿年 | bigint unsigned | 8 | 0 到约 10 的 19 次方 |

## Worked Example
优化查询 `select * from order where status = ? and create_time > ? order by create_time limit 100000, 20`：
1. 去掉 `*`，明确列出需要的字段（ORM 1）。
2. 建索引：status 是等号条件，create_time 是范围条件 → 等号条件放前面 → `idx_status_create_time`；order by 的字段刚好是索引的最后一列，可以利用有序性，避免 file_sort（索 5、索 9）。
3. 深分页 → 改成延迟关联：先 `select id ... limit 100000, 20`，再 join 回原表（索 7）。
4. 用 explain 验证 type 至少是 range（索 8）。
5. 分页前先执行 count，结果为 0 就直接返回（SQL 5）；参数用 `#{}` 传入（ORM 4）。

## Key Takeaways
1. 表必备 id、create_time、update_time；小数用 decimal；非负数用 unsigned；是否类字段用 `is_xxx tinyint`。
2. 业务上唯一的字段必须建唯一索引；join 不超过 3 个表；禁止左模糊；varchar 索引要指定长度。
3. 组合索引：等号列在前，区分度高的在前；用 explain 确认 type 至少为 range。
4. 用 count(*)；sum 要套 IFNULL；判断 NULL 用 ISNULL()。
5. 不用外键、级联和存储过程；订正数据前先 select。
6. ORM：不用 `*`、不用 `${}`、显式 resultMap、只更新改动过的字段。

## Connects To
- **ch01 / ch02**：POJO 布尔属性命名；DO 属性类型与字段类型匹配；BigDecimal 与 decimal 对应。
- **ch07**：SQL 注入防护。
- **ch10**：存储方案和数据结构的设计必须经过评审。
