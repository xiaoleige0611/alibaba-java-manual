# 第 9 章：工程结构 — 应用分层 / 二方库依赖 / 服务器

## Core Idea
工程结构决定了代码能不能长期维护：**分层**让每一层只做自己的事（包括异常和日志的职责划分），**分层领域模型**让对象在各层之间按职责流转，**二方库规约**用 GAV、语义化版本和仲裁一致性防止依赖地狱，**服务器规约**处理高并发下的 time_wait、文件句柄和 OOM 现场保留。

## Frameworks Introduced
- **应用分层**（上层依赖下层，也可以跨层直接依赖）：
  | 层 | 职责 |
  |---|---|
  | 开放接口层 | 把 Service 方法封装成 RPC 接口，或通过 Web 封装成 http 接口；做网关安全控制、流量控制 |
  | 终端显示层 | 各端的模板渲染和显示（velocity、JS、JSP、移动端） |
  | Web 层 | 访问控制转发、基本参数校验、不复用的简单业务处理 |
  | Service 层 | 相对具体的业务逻辑 |
  | **Manager 层** | 通用业务处理：① 封装第三方平台，预处理返回结果、转化异常；② Service 通用能力下沉（缓存方案、中间件处理）；③ 与 DAO 交互，组合复用多个 DAO |
  | DAO 层 | 数据访问（MySQL、Oracle、Hbase） |
  | 外部接口 / 第三方平台 | 其他部门的 RPC 接口、基础平台、其他公司的 HTTP 接口 |
- **分层异常处理规约**：
  | 层 | 做法 |
  |---|---|
  | DAO | `catch (Exception e)` 后 `throw new DAOException(e)`，**不打日志** |
  | Manager | 与 Service 同机部署时同 DAO 的做法；单独部署时同 Service 的做法 |
  | Service | **必须记录错误日志**，尽量带上参数（保护案发现场） |
  | Web | 绝不继续往上抛；跳转到友好的错误页面，给出用户能理解的提示 |
  | 开放接口 | 转换成**错误码加错误信息**返回 |
- **分层领域模型**：DO（与表结构一一对应，由 DAO 向上传输）→ DTO（Service 或 Manager 向外传输）→ BO（Service 输出的业务对象）→ AO（Web 与 Service 之间的应用对象，贴近展示层，复用度低）→ VO（Web 传给模板引擎的对象）；**Query**：各层接收上层查询请求的对象，**超过 2 个参数的查询必须封装，禁止用 Map 传输**。
- **GAV 规则**：GroupID 为 `com.{公司/BU}.业务线[.子业务线]`，最多 4 级；ArtifactID 为 `产品线名-模块名`（`dubbo-client`），先到中央仓库查重。
- **语义化版本**：`主版本号.次版本号.修订号`。主版本号：产品方向改变、大规模 API 不兼容、架构不兼容升级；次版本号：保持相对兼容，增加主要功能，允许影响范围极小的不兼容修改；修订号：完全兼容，修复 bug 或增加次要功能。**起始版本是 1.0.0**；正式版本号不允许覆盖。
- **二方库发布两原则**：精简可控（只包含 Service API、领域模型、Utils、常量、枚举；依赖其他二方库时尽量用 provided；不带日志实现，只依赖日志框架）；稳定可追溯（记录每个版本的变化、维护人、源码位置；用户不主动升级时行为不变）。

## 规约速查
| # | 级别 | 规则 |
|---|---|---|
| 分 1 | 推荐 | 按上面的七层结构分层 |
| 分 2 | 参考 | 分层异常处理规约 |
| 分 3 | 参考 | 分层领域模型；超过 2 个参数的查询禁止用 Map |
| 依 1 | 强制 | GAV 规则 |
| 依 2 | 强制 | 版本号格式 `主.次.修订`，起始版本 1.0.0 |
| 依 3 | 强制 | 线上应用不依赖 SNAPSHOT 版本（安全包除外），保证发布的幂等性 |
| 依 4 | 强制 | 新增或升级二方库时，除功能点以外的其他 jar 包仲裁结果保持不变；有变化就用 `dependency:resolve` 比对，用 `dependency:tree` 找差异，用 `<exclude>` 排除 |
| 依 5 | 强制 | 二方库可以定义枚举，参数可以用枚举，但**接口返回值不能是枚举或包含枚举的 POJO** |
| 依 6 | 强制 | 依赖同一组二方库时定义统一的版本变量（`${spring.version}`） |
| 依 7 | 强制 | 子项目 pom 中不能出现 GroupId、ArtifactId 相同但 Version 不同的依赖（本地调试正常，合并成 war 后线上出故障） |
| 依 8 | 推荐 | 底层基础框架、核心数据平台、近硬件端系统谨慎引入第三方实现 |
| 依 9 | 推荐 | 依赖声明放在 `<dependencies>`，版本仲裁放在 `<dependencyManagement>` |
| 依 10 | 推荐 | 二方库不要有配置项，至少不要再新增配置项 |
| 依 11 | 参考 | 二方库发布两原则 |
| 服 1 | 推荐 | 高并发服务器调小 time_wait 超时：`/etc/sysctl.conf` 中设置 `net.ipv4.tcp_fin_timeout = 30`（默认 240 秒） |
| 服 2 | 推荐 | 调大最大文件句柄数 fd（Linux 默认 1024，不够时报 “open too many files”） |
| 服 3 | 推荐 | JVM 加 `-XX:+HeapDumpOnOutOfMemoryError` 参数 |
| 服 4 | 推荐 | 线上 JVM 的 `Xms` 与 `Xmx` 设置相同，避免 GC 后调整堆大小 |
| 服 5 | 参考 | 服务器内部重定向用 forward；外部重定向地址用 URL 拼装工具类生成 |

## Key Concepts
- **一方库 / 二方库 / 三方库**：本工程子模块的 jar / 公司内部发布的 jar / 公司外部的开源 jar。
- **jar 包仲裁**：Maven 从多个版本中选出一个最终版本的机制。
- **SNAPSHOT**：快照版本，内容可变，发布结果不幂等。
- **Manager 层**：手册特有的通用业务处理层，位于 Service 与 DAO 之间。

## Mental Models
- **异常只记录一次，记录在离业务最近的地方**：DAO 只负责抛，Service 负责记，Web 负责兜底，开放接口负责翻译成错误码。
- **对象按层换名字**：DO 不出 DAO，VO 不进 Service；同一份数据在不同层用不同的对象承载。
- **二方库就是对外的承诺**：返回值里出现枚举，服务端一新增枚举值，老客户端反序列化就会失败。
- **把 OOM 当成需要保留现场的事故**：OOM 可能几个月才出现一次，没有 dump 就无从分析。

## Anti-patterns
- DAO 层打日志，Service 层再打一遍：浪费性能和存储。
- Web 层把异常继续往上抛：页面直接报错。
- 用 `Map<String, Object>` 传递 5 个查询参数：类型不可控，可读性差。
- 线上依赖 SNAPSHOT 版本：两次构建结果不同。
- 子模块各自写 spring 的版本号：线上只能留一个版本，结果线下调试正常、线上出故障。
- 二方库接口返回枚举：双方版本不一致时反序列化失败。

## Code Examples
```xml
<!-- 父 pom：只做版本仲裁 -->
<properties>
    <spring.version>5.1.8.RELEASE</spring.version>
</properties>
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
            <version>${spring.version}</version>
        </dependency>
    </dependencies>
</dependencyManagement>
<!-- 子 pom：显式声明依赖，version 和 scope 从父 pom 读取 -->
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-core</artifactId>
    </dependency>
</dependencies>
```
```bash
# 服务器调优
net.ipv4.tcp_fin_timeout = 30           # /etc/sysctl.conf
java -Xms4g -Xmx4g -XX:+HeapDumpOnOutOfMemoryError ...
```
- **What it demonstrates**：统一版本变量与 dependencyManagement 的分工；服务器规约的落地写法。（版本号为示意。）

## Worked Example
一个用户积分查询请求经过各层时的对象和异常流转：
1. 开放接口层接收 `PointQuery`（3 个参数，所以用 Query 对象，不用 Map）。
2. Service 层调用 Manager 层的 `PointManager`：Manager 封装了积分中心的 RPC（预处理返回结果、转化异常），并组合 `PointDAO` 和缓存。
3. DAO 层返回 `PointDO`；如果出现 SQLException，`throw new DAOException(e)`，不打日志。
4. Service 层捕获异常后执行 `logger.error("query point failed, query={}", query, e)`，再把 DO 组装成 `PointBO` 或 `PointDTO`。
5. 开放接口层把异常转换成 `errorCode` 和 `errorMsg` 返回；如果是 Web 页面，就由 Web 层组装 `PointVO` 交给模板，出错时跳转到友好的错误页面。
6. 积分中心的 client.jar 是二方库：版本 `1.2.0`，返回值中不含枚举，线上不依赖 SNAPSHOT 版本。

## Key Takeaways
1. 七层结构中，Manager 层负责封装第三方、下沉通用能力、组合多个 DAO。
2. 异常：DAO 只抛不记，Service 必须记，Web 不再往上抛，开放接口转成错误码。
3. 领域模型：DO / DTO / BO / AO / VO / Query 各司其职；超过 2 个参数的查询不用 Map。
4. 二方库：遵守 GAV 规则、语义化版本从 1.0.0 开始、线上不用 SNAPSHOT、返回值不含枚举、统一版本变量、升级时保持仲裁结果不变。
5. 服务器：调小 time_wait、调大 fd、加 HeapDumpOnOutOfMemoryError、Xms 与 Xmx 相同。

## Connects To
- **ch01**：领域模型和 Service/DAO 方法的命名。
- **ch06**：错误码 / 异常 / Result 三分法；日志规约。
- **ch04**：线程池的 OOM 风险与服务器的 OOM 现场保留相互配合。
