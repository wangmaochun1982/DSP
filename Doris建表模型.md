# Doris 数据表模型上目前分为三类： DUPLICATE KEY ， UNIQUE KEY ， AGGREGATE KEY 。

1.1 DUPLICATE KEY 表模型¶

<img width="755" height="496" alt="image" src="https://github.com/user-attachments/assets/416a9981-55e5-4299-b3a2-7d6711f0ce42" />


只指定排序列，相同的 KEY 行不会合并。

适用于数据无需提前聚合的分析业务：

原始数据分析

仅追加新数据的日志或时序数据分析

```sql
-- 例如 允许 KEY 重复仅追加新数据的日志数据分析
CREATE TABLE session_data
(
    visitorid   SMALLINT,
    sessionid   BIGINT,
    visittime   DATETIME,
    city        CHAR(20),
    province    CHAR(20),
    ip          varchar(32),
    brower      CHAR(20),
    url         VARCHAR(1024)
)
DUPLICATE KEY(visitorid, sessionid) -- 只用于指定排序列，相同的 KEY 行不会合并
DISTRIBUTED BY HASH(sessionid, visitorid) BUCKETS 10;
```


1.2 AGGREGATE KEY 表模型¶

<img width="754" height="446" alt="image" src="https://github.com/user-attachments/assets/b8ff7b78-58f0-4675-9701-c6fe53756ddf" />


AGGREGATE KEY 相同时，新旧记录进行聚合，目前支持的聚合方式：

SUM ：求和，多行的 Value 进行累加。

REPLACE ：替代，下一批数据中的 Value 会替换之前导入过的行中的 Value 。

MAX ：保留最大值。

MIN ：保留最小值。

REPLACE_IF_NOT_NULL ：非空值替换。和 REPLACE 的区别在于对于 null 值，不做替换。

HLL_UNION ： HLL 类型的列的聚合方式，通过 HyperLogLog 算法聚合。

BITMAP_UNION ： BIMTAP 类型的列的聚合方式，进行位图的并集聚合。

适合报表和多维分析业务：

网站流量分析

数据报表多维分析

```sql
-- 例如 网站流量分析
CREATE TABLE site_visit
(
    siteid      INT,
    city        SMALLINT,
    username    VARCHAR(32),
    pv BIGINT   SUM DEFAULT '0' -- PV 浏览量计算
)
AGGREGATE KEY(siteid, city, username) -- 相同的 KEY 行会合并，非 KEY 列会根据指定的聚合函数进行聚合
DISTRIBUTED BY HASH(siteid) BUCKETS 10;


1.3 UNIQUE KEY 表模型¶
UNIQUE KEY 相同时，新记录覆盖旧记录。在 1.2 版本之前， UNIQUE KEY 实现上和 AGGREGATE KEY 的 REPLACE 聚合方法一样，二者本质上相同，自 1.2 版本我们给 UNIQUE KEY 引入了 merge on write 实现，该实现有更好的聚合查询性能。

适用于有更新需求的分析业务：

订单去重分析

实时增删改同步


```sql

-- 例如 订单去重分析
CREATE TABLE sales_order
(
    orderid     BIGINT,
    status      TINYINT,
    username    VARCHAR(32),
    amount      BIGINT DEFAULT '0'
)
UNIQUE KEY(orderid) -- 相同的 KEY 行会合并
DISTRIBUTED BY HASH(orderid) BUCKETS 10;
```

 # 2 索引¶

索引用于帮助快速过滤或查找数据。目前主要支持两类索引：

内建自动创建的智能索引，包括前缀索引和 ZoneMap 索引。

用户手动创建的二级索引，包括倒排索引、 bloomfilter 索引、 ngram bloomfilter 索引和 bitmap 索引。

https://www.blueisacat.cn/documents/Doris/Doris%E7%94%A8%E6%88%B7%E6%89%8B%E5%86%8C/3%20%E6%95%B0%E6%8D%AE%E8%A1%A8%E8%AE%BE%E8%AE%A1/3.10%20%E6%95%B0%E6%8D%AE%E5%BA%93%E5%BB%BA%E8%A1%A8%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5/#3
 
