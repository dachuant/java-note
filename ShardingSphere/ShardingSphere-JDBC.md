# 1. 分库分表解决方案和代表框架

​	1.1. 客户端分片(ShardingSphere-JDBC)

​		<img src="https://gitee.com/dachuant/image/raw/master/picgo/20201213195128.png" alt="image" style="zoom:50%;" />

​	1.2. 代理服务器分片(Cobar、MyCat、ShardingSphere-Proxy)

​		<img src="https://gitee.com/dachuant/image/raw/master/picgo/20201213231826.png" alt="img" style="zoom:50%;" />

​	1.3 分布式数据库(TiDB)

# 2. ShardingSphere简介

​	Apache ShardingSphere 是一套开源的分布式数据库中间件解决方案组成的生态圈，它由 JDBC、Proxy 和 Sidecar（规划中）这 3 款相互独立，却又能够混合部署配合使用的产品组成。 它们均提供标准化的数据分片、分布式事务和数据库治理功能，可适用于如 Java 同构、异构语言、云原生等各种多样化的应用场景。（官网 http://shardingsphere.apache.org/index_zh.html）

# 3. ShardingSphere-JDBC简介

ShardingSphere-JDBC定位为轻量级 Java 框架，在 Java 的 JDBC 层提供的额外服务。 它使用客户端直连数据库，以 jar 包形式提供服务，无需额外部署和依赖，可理解为增强版的 JDBC 驱动，完全兼容 JDBC 和各种 ORM 框架。

- 适用于任何基于 JDBC 的 ORM 框架，如：JPA, Hibernate, Mybatis, Spring JDBC Template 或直接使用 JDBC。
- 支持任何第三方的数据库连接池，如：DBCP, C3P0, BoneCP, Druid, HikariCP 等。
- 支持任意实现 JDBC 规范的数据库，目前支持 MySQL，Oracle，SQLServer，PostgreSQL 以及任何遵循 SQL92 标准的数据库。

# 4. 分片算法

- 精确分片算法

  精确分片算法（PreciseShardingAlgorithm）用于单个字段作为分片键，SQL中有 `=` 与 `IN` 等条件的分片，需要在标准分片策略（`StandardShardingStrategy` ）下使用。

- 范围分片算法

  范围分片算法（RangeShardingAlgorithm）用于单个字段作为分片键，SQL中有 `BETWEEN AND`、`>`、`<`、`>=`、`<=` 等条件的分片，需要在标准分片策略（`StandardShardingStrategy` ）下使用。

- 复合分片算法

  复合分片算法（ComplexKeysShardingAlgorithm）用于多个字段作为分片键的分片操作，同时获取到多个分片健的值，根据多个字段处理业务逻辑。需要在复合分片策略（`ComplexShardingStrategy` ）下使用。

- Hint分片算法

  Hint分片算法（HintShardingAlgorithm）稍有不同，上边的算法中我们都是解析`SQL` 语句提取分片键，并设置分片策略进行分片。但有些时候我们并没有使用任何的分片键和分片策略，可还想将 SQL 路由到目标数据库和表，就需要通过手动干预指定SQL的目标数据库和表信息，这也叫强制路由。

# 5. 分片策略

​	分片策略只是抽象出的概念，它是由分片算法和分片健组合而成，分片算法做具体的数据分片逻辑。

- 标准分片策略

  标准分片策略适用于单分片键，此策略支持 `PreciseShardingAlgorithm` 和 `RangeShardingAlgorithm` 两个分片算法。

  其中 `PreciseShardingAlgorithm` 是必选的，用于处理 `=` 和 `IN` 的分片。`RangeShardingAlgorithm` 是可选的，用于处理`BETWEEN AND`， `>`， `<`，`>=`，`<=` 条件分片，如果不配置`RangeShardingAlgorithm`，SQL中的条件等将按照全库路由处理。

- 复合分片策略

  复合分片策略，同样支持对 SQL语句中的 `=`，`>`， `<`， `>=`， `<=`，`IN`和 `BETWEEN AND` 的分片操作。不同的是它支持多分片键，具体分配片细节完全由应用开发者实现。

- 行表达式分片策略

  行表达式分片策略，支持对 SQL语句中的 `=` 和 `IN` 的分片操作，但只支持单分片键。这种策略通常用于简单的分片，不需要自定义分片算法，可以直接在配置文件中接着写规则。

  `t_order_$->{t_order_id % 4}` 代表 `t_order` 对其字段 `t_order_id`取模，拆分成4张表，而表名分别是`t_order_0` 到 `t_order_3`。

- Hint分片策略

  Hint分片策略，对应上边的Hint分片算法，通过指定分片健而非从 `SQL`中提取分片健的方式进行分片的策略。

# 6. 绑定表和广播表

绑定表（BindingTable）：是指与分片规则一致的一组主表和子表。

广播表（BroadCastTable）：是指所有分片数据源中都存在的表，也就是说，这种表的表结构和表中的数据在每个数据库中都是完全一样的。









https://mp.weixin.qq.com/s?__biz=MzAxNTM4NzAyNg==&mid=2247488500&idx=1&sn=108bf704a54b0a9638e84698deb3ce4c&chksm=9b858309acf20a1fc606f6d140e9638072405011829bb8decc906a648d3f2f75441c0adac869&token=1691474648&lang=zh_CN#rd

https://blog.csdn.net/xinzhifu1/article/details/110442458

