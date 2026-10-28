# 第 3 章：编程规约 —（五）集合处理

## Core Idea
Java 集合 API 中很多返回值是**视图**或**不可变对象**（subList、keySet、Arrays.asList、Collections.emptyList），看起来像普通 List 或 Set，修改时却会抛异常，或者悄悄影响原集合。这一章教你识别这些“伪集合”，并正确处理 hashCode/equals、toArray、foreach 删除、Comparator 和容量。

## Frameworks Introduced
- **hashCode/equals 三条铁律**：覆写 equals 就必须覆写 hashCode；放进 Set 的对象必须覆写这两个方法；作为 Map key 的自定义对象必须覆写这两个方法（String 已经覆写，可以放心当 key）。
- **toArray 的长度选择**：必须用 `toArray(T[] array)`，传入类型一致、**长度为 0** 的空数组。
  | 传入数组的 length | 结果 |
  |---|---|
  | = 0 | 动态创建与 size 相同的数组，**性能最好** |
  | 0 < length < size | 重新创建数组，增加 GC 负担 |
  | = size | 高并发下 size 变大后，问题同上 |
  | > size | 浪费空间，并在 size 位置插入 null，有 NPE 隐患 |
- **PECS（Producer Extends Consumer Super）**：主要往外读取的用 `<? extends T>`（不能 add）；主要往里插入的用 `<? super T>`（不能 get）。
- **Comparator 三条件**（JDK7+，不满足时 `Arrays.sort` 和 `Collections.sort` 会抛 IllegalArgumentException）：自反对称（x,y 的比较结果与 y,x 相反）、传递（x>y、y>z 则 x>z）、相等一致（x=y 时，x,z 与 y,z 的比较结果相同）。
- **HashMap 初始容量公式**：`initialCapacity = (需要存储的元素个数 / 负载因子) + 1`，负载因子默认 0.75；暂时无法确定时设为 16。
- **有序性与稳定性**：sort（遍历结果按比较规则排列）与 order（每次遍历的次序固定）是两回事。ArrayList 是 order/unsort，HashMap 是 unorder/unsort，TreeSet 是 order/sort。

## 规约速查
> 编号沿用华山版；标 `泰 x.y` 的行是泰山版（v1.6.0）新增条款，数字为泰山版（六）集合处理中的编号（泰山版插入新条款后，后续编号整体后移）。

| # | 级别 | 规则 |
|---|---|---|
| 1 | 强制 | hashCode/equals 三条铁律 |
| 泰 2 | 强制 | **判断集合是否为空用 `isEmpty()`，不用 `size() == 0`**（O(1)，可读性更好） |
| 泰 3 | 强制 | `Collectors.toMap()` 必须用带 `mergeFunction`（BinaryOperator）的重载，否则 key 重复时抛 IllegalStateException |
| 泰 4 | 强制 | `Collectors.toMap()` 的 value 为 null 时抛 NPE（`HashMap.merge` 内部判空） |
| 2 | 强制 | `subList` 的结果不能强转成 ArrayList（它是 SubList 视图，会抛 ClassCastException）。原理见 ch13 第 5 问 |
| 3 | 强制 | `keySet()/values()/entrySet()` 返回的集合不能添加元素 |
| 4 | 强制 | `Collections.emptyList()/singletonList()` 不可变，不能增删 |
| 5 | 强制 | 使用 subList 期间修改原集合，子列表的操作会抛 ConcurrentModificationException。原理见 ch13 第 5 问 |
| 6 | 强制 | 集合转数组用 `toArray(new T[0])` |
| 7 | 强制 | 调用 `addAll()` 前先判断参数是否为 null |
| 8 | 强制 | `Arrays.asList()` 的结果不能 add/remove/clear；修改原数组会同步影响 List |
| 9 | 强制 | `<? extends T>` 不能 add，`<? super T>` 不能 get |
| 10 | 强制 | 把无泛型集合赋值给泛型集合后，使用元素前先做 instanceof 判断 |
| 11 | 强制 | **不要在 foreach 循环里 remove/add**；删除用 Iterator，并发场景要给 Iterator 加锁。原理见 ch13 第 7 问 |
| 12 | 强制 | Comparator 满足三条件（要处理相等的情况） |
| 13 | 推荐 | 使用 diamond 语法 `new HashMap<>(16)` |
| 14 | 推荐 | 集合初始化时指定容量。原理见 ch13 第 2、3 问 |
| 15 | 推荐 | 遍历 Map 用 entrySet，JDK8 用 `Map.forEach` |
| 16 | 推荐 | 注意各种 Map 的 K/V 能否为 null（见下表） |
| 17 | 参考 | 利用集合的有序性和稳定性 |
| 18 | 参考 | 用 Set 去重，不要用 List.contains 遍历去重 |

## Key Concepts
- **视图（View）**：subList、keySet、Arrays.asList 返回的对象，背后的数据仍是原集合或原数组。
- **适配器模式**：Arrays.asList 只转换了接口，后台数据仍然是数组。
- **immutable list**：Collections 工具类返回的不可变集合。
- **fail-fast / ConcurrentModificationException**：遍历期间结构被修改时快速失败。
- **resize**：HashMap 扩容，需要重建 hash 表。

## Mental Models
- **拿到一个集合，先问“它是谁的”**：如果是别人对象的视图或不可变对象，就复制一份再修改：`new ArrayList<>(list.subList(...))`。
- **遍历时删除 = 用 Iterator 删除**：foreach 删除能不能成功取决于元素位置（删 "1" 不报错，删 "2" 会抛异常），这是在碰运气。
- **容量要事先算好**：放 1024 个元素却不指定容量，HashMap 会被迫扩容 7 次。

## Anti-patterns
- 查询无结果时返回 `Collections.emptyList()`，调用方往里 add，抛 UnsupportedOperationException。
- `(ArrayList) list.subList(0, 2)` 抛 ClassCastException。
- 以为 ConcurrentHashMap 能像 HashMap 一样存 null，结果抛 NPE。
- Comparator 写成 `o1.getId() > o2.getId() ? 1 : -1`，没处理相等的情况。
- 用 `keySet` 遍历再 `get`：实际遍历了两次。

## Code Examples
```java
// 正例：用 Iterator 删除
Iterator<String> iterator = list.iterator();
while (iterator.hasNext()) {
    String item = iterator.next();
    if (删除元素的条件) {
        iterator.remove();
    }
}
// 反例：结果取决于删除的是哪个元素（"1" 正常，"2" 抛 ConcurrentModificationException）
for (String item : list) {
    if ("1".equals(item)) {
        list.remove(item);
    }
}

// 正例：toArray
String[] array = list.toArray(new String[0]);

// Arrays.asList 的两个坑
String[] str = new String[] { "yang", "hao" };
List list = Arrays.asList(str);
list.add("yangguanbao");   // 运行时抛 UnsupportedOperationException
str[0] = "changed";        // list.get(0) 也随之变化，反之亦然

// 泰山新增：toMap 的两个坑
Map<String, Double> map = pairs.stream().collect(
    Collectors.toMap(Pair::getKey, Pair::getValue, (v1, v2) -> v2));  // 重复 key 保留后者
// 不传 mergeFunction 且 key 重复 → IllegalStateException；value 为 null → NPE

// 反例：无泛型集合赋值给泛型集合
List<String> generics = null;
List notGenerics = new ArrayList(10);
notGenerics.add(new Object());
generics = notGenerics;
String string = generics.get(0);   // 抛 ClassCastException
```
- **What it demonstrates**：视图、不可变集合、泛型擦除带来的运行时异常。

## Reference Tables
| 集合类 | Key | Value | Super | 说明 |
|---|---|---|---|---|
| Hashtable | 不允许 null | 不允许 null | Dictionary | 线程安全 |
| ConcurrentHashMap | 不允许 null | 不允许 null | AbstractMap | 锁分段技术（JDK8 用 CAS） |
| TreeMap | 不允许 null | 允许 null | AbstractMap | 线程不安全 |
| HashMap | 允许 null | 允许 null | AbstractMap | 线程不安全 |

## Worked Example
需求：给一批订单按 id 排序后去重，再转成数组交给下游。
1. 去重：`Set<Long> ids = new HashSet<>((int) (n / 0.75f) + 1);`（第 14、18 条：用 Set 去重，并指定容量）。
2. 订单对象要放进 Set，所以 `Order` 必须同时覆写 hashCode 和 equals（第 1 条）。
3. 排序：用 `Comparator.comparing(Order::getId)`，或者手写时处理相等的情况：`Long.compare(o1.getId(), o2.getId())`（第 12 条）。
4. 只取前 10 条：`new ArrayList<>(sorted.subList(0, 10))`，复制一份，避免持有视图（第 2、5 条）。
5. 转数组：`orders.toArray(new Order[0])`（第 6 条）。
6. 如果需要合并上游返回的列表：`if (upstream != null) result.addAll(upstream);`（第 7 条）。

## Key Takeaways
1. subList、keySet、asList、emptyList 都不是可以随意修改的普通集合，要修改就先复制。
2. 覆写 equals 必须同时覆写 hashCode；作为 Set 元素或 Map key 的对象必须覆写这两个方法。
3. `toArray(new T[0])` 是性能最好的写法。
4. 遍历时删除元素一律用 Iterator。
5. 容量按 `n / 0.75 + 1` 计算；遍历 Map 用 entrySet 或 forEach。
6. ConcurrentHashMap 和 Hashtable 的 key 和 value 都不能为 null。
7. （泰山）判空用 `isEmpty()`；`Collectors.toMap` 一律传 mergeFunction，并先过滤 null value。

## Connects To
- **ch04**：HashMap 在高并发 resize 时可能出现死链；并发修改集合的问题。
- **ch05**：“任何数据结构都应指定大小”。
- **ch08**：iBATIS 的 queryForList 内部用 subList 做分页，是性能陷阱。
- **ch13**：第 2、3 问（HashMap 容量与扩容）、第 5 问（subList 视图）、第 7 问（modCount 与 fail-fast）。
