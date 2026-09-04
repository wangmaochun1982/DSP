# 1 Row & Column¶

在 Doris 中，数据都以表（ Table ）的形式进行逻辑上的描述。

一张表包括行（ Row ）和列（ Column ）：

Row ：即用户的一行数据；

Column ：用于描述一行数据中不同的字段；

Column 可以分为两大类： Key 和 Value 。

从业务角度看， Key 和 Value 可以分别对应维度列和指标列。 

Doris 的 key 列是建表语句中指定的列，建表语句中的关键字 'unique key' 或 'aggregate key' 或 'duplicate key' 后面的列就是 key 列，除了 key 列剩下的就是 value 列。

从聚合模型的角度来说， Key 列相同的行，会聚合成一行。其中 Value 列的聚合方式由用户在建表时指定。

关于更多聚合模型的介绍，可以参阅 Doris 数据模型。


# 2 分区和分桶（Partition & Bucket）¶

> 分区是指根据表中特定列的值进行分段，将表中的数据划分为更小，更易于管理的不相交的子集，每一个数据子集称为一个分区。每一行数据属于且只属于一个特定的分区。分区可以视为最小的逻辑管理单元。

目前 Doris 支持 Range 和 List 的分区划分方式。建表时如果不指定分区，此时 Doris 会生成一个默认的分区包含表中的所有数据，这个分区对用户是透明的。

> 分桶¶
分桶是指将一个分区中的数据进一步按照某种规则被划分更小的不相交的数据单元。每一行数据属于且只属于一个特定的分桶。与根据特定列值对数据进行分段的分区不同，分桶尝试将数据均匀分布在预定义的桶中，从而减少数据倾斜的问题。
>
> 分桶通过确保数据分布均匀并提高数据局部性以提升查询执行的性能。

目前 Doris 支持 Hash 和 Random 的分桶划分方式。

一个分桶在物理上对应一个数据分片（ Tablet ），数据分片在物理上是独立存储的，它是数据移动、复制等操作的最小物理存储单元。


这里给出了一个采用了 Range 分区和 Hash 分桶的建表举例。

```sql
-- Range Partition
CREATE TABLE IF NOT EXISTS example_range_tbl
(
    `user_id` LARGEINT NOT NULL COMMENT "用户id",
    `date` DATE NOT NULL COMMENT "数据灌入日期时间",
    `timestamp` DATETIME NOT NULL COMMENT "数据灌入的时间戳",
    `city` VARCHAR(20) COMMENT "用户所在城市",
    `age` SMALLINT COMMENT "用户年龄",
    `sex` TINYINT COMMENT "用户性别",
    `last_visit_date` DATETIME REPLACE DEFAULT "1970-01-01 00:00:00" COMMENT "用户最后一次访问时间",
    `cost` BIGINT SUM DEFAULT "0" COMMENT "用户总消费",
    `max_dwell_time` INT MAX DEFAULT "0" COMMENT "用户最大停留时间",
    `min_dwell_time` INT MIN DEFAULT "99999" COMMENT "用户最小停留时间"
)
ENGINE=OLAP
AGGREGATE KEY(`user_id`, `date`, `timestamp`, `city`, `age`, `sex`)
PARTITION BY RANGE(`date`)
(
    PARTITION `p201701` VALUES LESS THAN ("2017-02-01"),
    PARTITION `p201702` VALUES LESS THAN ("2017-03-01"),
    PARTITION `p201703` VALUES LESS THAN ("2017-04-01"),
    PARTITION `p2018` VALUES [("2018-01-01"), ("2019-01-01"))
)
DISTRIBUTED BY HASH(`user_id`) BUCKETS 16
PROPERTIES
(
    "replication_num" = "1"
);


这里以 AGGREGATE KEY 数据模型为例进行说明。 AGGREGATE KEY 数据模型中，所有没有指定聚合方式（ SUM 、 REPLACE 、 MAX 、 MIN ）的列视为 Key 列。而其余则为 Value 列。

在建表语句的最后 PROPERTIES 中，关于 PROPERTIES 中可以设置的相关参数，可以查看 CREATE TABLE 中的详细介绍。

ENGINE 的类型是 OLAP ，即默认的 ENGINE 类型。在 Doris 中，只有这个 ENGINE 类型是由 Doris 负责数据管理和存储的。

其他 ENGINE 类型，如 MySQL 、 Broker 、 ES 等等，本质上只是对外部其他数据库或系统中的表的映射，以保证 Doris 可以读取这些数据。

而 Doris 本身并不创建、管理和存储任何非 OLAP ENGINE 类型的表和数据。

IF NOT EXISTS 表示如果没有创建过该表，则创建。注意这里只判断表名是否存在，而不会判断新建表 Schema 是否与已存在的表 Schema 相同。

所以如果存在一个同名但不同 Schema 的表，该命令也会返回成功，但并不代表已经创建了新的表和新的 Schema 。
```

# 4 查看分区信息¶
可以通过 show create table 来查看表的分区信息。

```sql

> show create table  example_range_tbl 
+-------------------+---------------------------------------------------------------------------------------------------------+                                                                                                            
| Table             | Create Table                                                                                            |                                                                                                            
+-------------------+---------------------------------------------------------------------------------------------------------+                                                                                                            
| example_range_tbl | CREATE TABLE `example_range_tbl` (                                                                      |                                                                                                            
|                   |   `user_id` largeint(40) NOT NULL COMMENT '用户id',                                                     |                                                                                                            
|                   |   `date` date NOT NULL COMMENT '数据灌入日期时间',                                                      |                                                                                                            
|                   |   `timestamp` datetime NOT NULL COMMENT '数据灌入的时间戳',                                             |                                                                                                            
|                   |   `city` varchar(20) NULL COMMENT '用户所在城市',                                                       |                                                                                                            
|                   |   `age` smallint(6) NULL COMMENT '用户年龄',                                                            |                                                                                                            
|                   |   `sex` tinyint(4) NULL COMMENT '用户性别',                                                             |                                                                                                            
|                   |   `last_visit_date` datetime REPLACE NULL DEFAULT "1970-01-01 00:00:00" COMMENT '用户最后一次访问时间', |                                                                                                            
|                   |   `cost` bigint(20) SUM NULL DEFAULT "0" COMMENT '用户总消费',                                          |                                                                                                            
|                   |   `max_dwell_time` int(11) MAX NULL DEFAULT "0" COMMENT '用户最大停留时间',                             |                                                                                                            
|                   |   `min_dwell_time` int(11) MIN NULL DEFAULT "99999" COMMENT '用户最小停留时间'                          |                                                                                                            
|                   | ) ENGINE=OLAP                                                                                           |                                                                                                            
|                   | AGGREGATE KEY(`user_id`, `date`, `timestamp`, `city`, `age`, `sex`)                                     |                                                                                                            
|                   | COMMENT 'OLAP'                                                                                          |                                                                                                            
|                   | PARTITION BY RANGE(`date`)                                                                              |                                                                                                            
|                   | (PARTITION p201701 VALUES [('0000-01-01'), ('2017-02-01')),                                             |                                                                                                            
|                   | PARTITION p201702 VALUES [('2017-02-01'), ('2017-03-01')),                                              |                                                                                                            
|                   | PARTITION p201703 VALUES [('2017-03-01'), ('2017-04-01')))                                              |                                                                                                            
|                   | DISTRIBUTED BY HASH(`user_id`) BUCKETS 16                                                               |                                                                                                            
|                   | PROPERTIES (                                                                                            |                                                                                                            
|                   | "replication_allocation" = "tag.location.default: 1",                                                   |                                                                                                            
|                   | "is_being_synced" = "false",                                                                            |                                                                                                            
|                   | "storage_format" = "V2",                                                                                |                                                                                                            
|                   | "light_schema_change" = "true",                                                                         |                                                                                                            
|                   | "disable_auto_compaction" = "false",                                                                    |                                                                                                            
|                   | "enable_single_replica_compaction" = "false"                                                            |                                                                                                            
|                   | );                                                                                                      |                                                                                                            
+-------------------+---------------------------------------------------------------------------------------------------------+

可以通过 show partitions from your_table 来查看表的分区信息。

> show partitions from example_range_tbl
+-------------+---------------+----------------+---------------------+--------+--------------+--------------------------------------------------------------------------------+-----------------+---------+----------------+---------------
+---------------------+---------------------+--------------------------+----------+------------+-------------------------+-----------+                                                                                                     
| PartitionId | PartitionName | VisibleVersion | VisibleVersionTime  | State  | PartitionKey | Range                                                                          | DistributionKey | Buckets | ReplicationNum | StorageMedium 
| CooldownTime        | RemoteStoragePolicy | LastConsistencyCheckTime | DataSize | IsInMemory | ReplicaAllocation       | IsMutable |                                                                                                     
+-------------+---------------+----------------+---------------------+--------+--------------+--------------------------------------------------------------------------------+-----------------+---------+----------------+---------------
+---------------------+---------------------+--------------------------+----------+------------+-------------------------+-----------+                                                                                                     
| 28731       | p201701       | 1              | 2024-01-25 10:50:51 | NORMAL | date         | [types: [DATEV2]; keys: [0000-01-01]; ..types: [DATEV2]; keys: [2017-02-01]; ) | user_id         | 16      | 1              | HDD           
| 9999-12-31 23:59:59 |                     |                    | 0.000    | false      | tag.location.default: 1 | true      |                                                                                                     
| 28732       | p201702       | 1              | 2024-01-25 10:50:51 | NORMAL | date         | [types: [DATEV2]; keys: [2017-02-01]; ..types: [DATEV2]; keys: [2017-03-01]; ) | user_id         | 16      | 1              | HDD           
| 9999-12-31 23:59:59 |                     |                    | 0.000    | false      | tag.location.default: 1 | true      |                                                                                                     
| 28733       | p201703       | 1              | 2024-01-25 10:50:51 | NORMAL | date         | [types: [DATEV2]; keys: [2017-03-01]; ..types: [DATEV2]; keys: [2017-04-01]; ) | user_id         | 16      | 1              | HDD           
| 9999-12-31 23:59:59 |                     |                    | 0.000    | false      | tag.location.default: 1 | true      |                                                                                                     
+-------------+---------------+----------------+---------------------+--------+--------------+--------------------------------------------------------------------------------+-----------------+---------+----------------+---------------
+---------------------+---------------------+--------------------------+----------+------------+-------------------------+-----------+

5 修改分区信息¶


通过 alter table add partition 来增加新的分区

ALTER TABLE example_range_tbl ADD  PARTITION p201704 VALUES LESS THAN("2020-05-01") DISTRIBUTED BY HASH(`user_id`) BUCKETS 5;
```


# 3.7.2 手动分区¶

Range 分区¶

分区列通常为时间列，以方便的管理新旧数据。 Range 分区支持的列类型 DATE ， DATETIME ， TINYINT ， SMALLINT ， INT ， BIGINT ， LARGEINT 。

1.FIXED RANGE ：定义分区的左闭右开区间。

```sql
PARTITION BY RANGE(col1[, col2, ...]) 
(
    PARTITION partition_name1 VALUES [("k1-lower1", "k2-lower1", "k3-lower1",...), ("k1-upper1", "k2-upper1", "k3-upper1", ...)),     
    PARTITION partition_name2 VALUES [("k1-lower1-2", "k2-lower1-2", ...), ("k1-upper1-2", MAXVALUE, ))
)


示例如下：



PARTITION BY RANGE(`date`)
(
    PARTITION `p201701` VALUES [("2017-01-01"),  ("2017-02-01")),
    PARTITION `p201702` VALUES [("2017-02-01"), ("2017-03-01")),
    PARTITION `p201703` VALUES [("2017-03-01"), ("2017-04-01"))
)
```

2.LESS THAN ：仅定义分区上界。下界由上一个分区的上界决定。

```sql

PARTITION BY RANGE(col1[, col2, ...])   
(
    PARTITION partition_name1 VALUES LESS THAN MAXVALUE | ("value1", "value2", ...),    
    PARTITION partition_name2 VALUES LESS THAN MAXVALUE | ("value1", "value2", ...)
)


PARTITION BY RANGE(`date`)
(
    PARTITION `p201701` VALUES LESS THAN ("2017-02-01"),
    PARTITION `p201702` VALUES LESS THAN ("2017-03-01"),
    PARTITION `p201703` VALUES LESS THAN ("2017-04-01"),
    PARTITION `p2018` VALUES [("2018-01-01"), ("2019-01-01")),
    PARTITION `other` VALUES LESS THAN (MAXVALUE)
)
```

3.BATCH RANGE ：批量创建数字类型和时间类型的 RANGE 分区，定义分区的左闭右开区间，设定步长。

```sql
PARTITION BY RANGE(int_col)
(
    FROM (start_num) TO (end_num) INTERVAL interval_value
)

PARTITION BY RANGE(date_col)      
(
    FROM ("start_date") TO ("end_date") INTERVAL num YEAR | num MONTH | num WEEK | num DAY ｜ 1 HOUR
)



PARTITION BY RANGE(age)
(
    FROM (1) TO (100) INTERVAL 10
)

PARTITION BY RANGE(`date`)
(
    FROM ("2000-11-14") TO ("2021-11-14") INTERVAL 2 YEAR
)

```

4.MULTI RANGE ：批量创建 RANGE 分区，定义分区的左闭右开区间。示例如下：

```sql
PARTITION BY RANGE(col)
(
FROM ("2000-11-14") TO ("2021-11-14") INTERVAL 1 YEAR,
FROM ("2021-11-14") TO ("2022-11-14") INTERVAL 1 MONTH,
FROM ("2022-11-14") TO ("2023-01-03") INTERVAL 1 WEEK,
FROM ("2023-01-03") TO ("2023-01-14") INTERVAL 1 DAY,
PARTITION p_20230114 VALUES [('2023-01-14'), ('2023-01-15'))
)
```


# 3.7.3 动态分区¶
动态分区只支持在 DATE/DATETIME 列上进行 Range 类型的分区。

建表时指定

```sql

CREATE TABLE tbl1
(...)
PROPERTIES
(
    "dynamic_partition.prop1" = "value1",
    "dynamic_partition.prop2" = "value2",
    ...
)
```

运行时修改

```sql
ALTER TABLE tbl1 SET
(
    "dynamic_partition.prop1" = "value1",
    "dynamic_partition.prop2" = "value2",
    ...
)

```

示例1：表 tbl1 分区列 k1 类型为 DATE ，创建一个动态分区规则。按天分区，只保留最近 7 天的分区，并且预先创建未来 3 天的分区。

```sql
CREATE TABLE tbl1
(
    k1 DATE,
    ...
)
PARTITION BY RANGE(k1) ()
DISTRIBUTED BY HASH(k1)
PROPERTIES
(
    "dynamic_partition.enable" = "true",
    "dynamic_partition.time_unit" = "DAY",
    "dynamic_partition.start" = "-7",
    "dynamic_partition.end" = "3",
    "dynamic_partition.prefix" = "p",
    "dynamic_partition.buckets" = "32"
);
```

假设当前日期为 2020-05-29 。则根据以上规则， tbl1 会产生以下分区：

```sql
p20200529: ["2020-05-29", "2020-05-30")
p20200530: ["2020-05-30", "2020-05-31")
p20200531: ["2020-05-31", "2020-06-01")
p20200601: ["2020-06-01", "2020-06-02")
```

在第二天，即 2020-05-30 ，会创建新的分区 p20200602: ["2020-06-02", "2020-06-03")

在 2020-06-06 时，因为 dynamic_partition.start 设置为 7 ，则将删除 7 天前的分区，即删除分区 p20200529 。


示例2： 表 tbl1 分区列 k1 类型为 DATETIME ，创建一个动态分区规则。按星期分区，只保留最近 2 个星期的分区，并且预先创建未来 2 个星期的分区。

```sql
CREATE TABLE tbl1
(
    k1 DATETIME,
    ...
)
PARTITION BY RANGE(k1) ()
DISTRIBUTED BY HASH(k1)
PROPERTIES
(
    "dynamic_partition.enable" = "true",
    "dynamic_partition.time_unit" = "WEEK",
    "dynamic_partition.start" = "-2",
    "dynamic_partition.end" = "2",
    "dynamic_partition.prefix" = "p",
    "dynamic_partition.buckets" = "8"
);

```


假设当前日期为 2020-05-29 ，是 2020 年的第 22 周。默认每周起始为星期一。则以上规则， tbl1 会产生以下分区：

```SQL

p2020_22: ["2020-05-25 00:00:00", "2020-06-01 00:00:00")
p2020_23: ["2020-06-01 00:00:00", "2020-06-08 00:00:00")
p2020_24: ["2020-06-08 00:00:00", "2020-06-15 00:00:00")

```
其中每个分区的起始日期为当周的周一。同时，因为分区列 k1 的类型为 DATETIME ，则分区值会补全时分秒部分，且皆为 0 。

在 2020-06-15 ，即第 25 周时，会删除 2 周前的分区，即删除 p2020_22 。

在上面的例子中，假设用户指定了周起始日为 "dynamic_partition.start_day_of_week" = "3" ，即以每周三为起始日。则分区如下：

```SQL

p2020_22: ["2020-05-27 00:00:00", "2020-06-03 00:00:00")
p2020_23: ["2020-06-03 00:00:00", "2020-06-10 00:00:00")
p2020_24: ["2020-06-10 00:00:00", "2020-06-17 00:00:00")
即分区范围为当周的周三到下周的周二。
```

Warning

注： 2019-12-31 和 2020-01-01 在同一周内，如果分区的起始日期为 2019-12-31 ，则分区名为 p2019_53 ，如果分区的起始日期为 2020-01-01 ，则分区名为 p2020_01 。


示例3：表 tbl1 分区列 k1 类型为 DATE ，创建一个动态分区规则。按月分区，不删除历史分区，并且预先创建未来 2 个月的分区。同时设定以每月 3 号为起始日。

```sql
CREATE TABLE tbl1
(
    k1 DATE,
    ...
)
PARTITION BY RANGE(k1) ()
DISTRIBUTED BY HASH(k1)
PROPERTIES
(
    "dynamic_partition.enable" = "true",
    "dynamic_partition.time_unit" = "MONTH",
    "dynamic_partition.end" = "2",
    "dynamic_partition.prefix" = "p",
    "dynamic_partition.buckets" = "8",
    "dynamic_partition.start_day_of_month" = "3"
);
```
假设当前日期为 2020-05-29 。则基于以上规则， tbl1 会产生以下分区：

```SQL

p202005: ["2020-05-03", "2020-06-03")
p202006: ["2020-06-03", "2020-07-03")
p202007: ["2020-07-03", "2020-08-03")

```
因为没有设置 dynamic_partition.start ，则不会删除历史分区。

假设今天为 2020-05-20 ，并设置以每月 28 号为起始日，则分区范围为：

```SQL

p202004: ["2020-04-28", "2020-05-28")
p202005: ["2020-05-28", "2020-06-28")
p202006: ["2020-06-28", "2020-07-28")

```



```text

5 原理与控制行为¶
Doris FE 中有固定的 dynamic partition 控制线程，持续以特定时间间隔（即 dynamic_partition_check_interval_seconds ）进行 dynamic partition 表的分区检查，完成需要的分区创建与删除操作。

具体而言，自动分区将会进行如下检查与操作（我们称此时该表分区的起始包含时间为 START ，末尾包含时间为 END ，省略 property 的 dynamic_partition. 前缀）：

START 时间之前的所有分区，全部被删除。

如果 create_history_partition 为 false ，创建当前时间到 END 之间的所有分区；如果 create_history_partition 为 true ，除当前时间到 END 之间的分区外，还会创建 START 到当前时间的分区。若定义了 history_partition_num ，则从当前时间向前创建的分区数量不超过 history_partition_num 。

需要注意的是：

如果分区时间范围与 [START, END] 范围相交，则认为属于当前 dynamic partition 时间范围。

如果尝试创建的新分区和现有分区冲突，则保留当前分区，不创建该新分区。如果该行为出现在建表时， DDL 将会报错。

因此，自动分区表在系统自动维护后，呈现的状态是：

START 时间之前，除 reserved_history_periods 所指定范围以外，不包含任何分区；

END 时间之后，保留所有手动创建的分区。

除手动删除或意外丢失的分区外，表包含特定范围内的全部分区：

如果 create_history_partition 为 true ：

若定义了 history_partition_num ，则特定范围为 [max(START, 当前时间 - history_partition_num * time_unit), END] ；

若未定义 history_partition_num ，则特定范围为 [START, END] ；

如果 create_history_partition 为 false ，则特定范围为 [当前时间, END] ，同时包含 [START, 当前时间) 中既存的分区。

整个特定范围按照 time_unit 划分为若干分区范围。对于任意一个范围，如果其与某个当前存在的分区 X 相交，则 X 被保留，否则该范围将被 dynamic partition 所创建的一个分区所完整覆盖。

除非分区数量即将超过 max_dynamic_partition_num ，创建将会失败。
```


7 查看动态分区表调度情况¶
通过以下命令可以进一步查看当前数据库下，所有动态分区表的调度情况：

```sql
> SHOW DYNAMIC PARTITION TABLES;
+-----------+--------+----------+-------------+------+--------+---------+-----------+----------------+---------------------+--------+------------------------+----------------------+-------------------------+
| TableName | Enable | TimeUnit | Start       | End  | Prefix | Buckets | StartOf   | LastUpdateTime | LastSchedulerTime   | State  | LastCreatePartitionMsg | LastDropPartitionMsg | ReservedHistoryPeriods  |
+-----------+--------+----------+-------------+------+--------+---------+-----------+----------------+---------------------+--------+------------------------+----------------------+-------------------------+
| d3        | true   | WEEK     | -3          | 3    | p      | 1       | MONDAY    | N/A            | 2020-05-25 14:29:24 | NORMAL | N/A                    | N/A                  | [2021-12-01,2021-12-31] |
| d5        | true   | DAY      | -7          | 3    | p      | 32      | N/A       | N/A            | 2020-05-25 14:29:24 | NORMAL | N/A                    | N/A                  | NULL                    |
| d4        | true   | WEEK     | -3          | 3    | p      | 1       | WEDNESDAY | N/A            | 2020-05-25 14:29:24 | NORMAL | N/A                    | N/A                  | NULL                    | 
| d6        | true   | MONTH    | -2147483648 | 2    | p      | 8       | 3rd       | N/A            | 2020-05-25 14:29:24 | NORMAL | N/A                    | N/A                  | NULL                    |
| d2        | true   | DAY      | -3          | 3    | p      | 32      | N/A       | N/A            | 2020-05-25 14:29:24 | NORMAL | N/A                    | N/A                  | NULL                    |
| d7        | true   | MONTH    | -2147483648 | 5    | p      | 8       | 24th      | N/A            | 2020-05-25 14:29:24 | NORMAL | N/A                    | N/A                  | NULL                    |
+-----------+--------+----------+-------------+------+--------+---------+-----------+----------------+---------------------+--------+------------------------+----------------------+-------------------------+
7 rows in set (0.02 sec)





```


# 3.7.4 自动分区¶


```sql
CREATE TABLE `DAILY_TRADE_VALUE`
(
    `TRADE_DATE`              datev2 NOT NULL COMMENT '交易日期',
    `TRADE_ID`                varchar(40) NOT NULL COMMENT '交易编号',
    ......
)
UNIQUE KEY(`TRADE_DATE`, `TRADE_ID`)
PARTITION BY RANGE(`TRADE_DATE`)
(
    PARTITION p_2000 VALUES [('2000-01-01'), ('2001-01-01')),
    PARTITION p_2001 VALUES [('2001-01-01'), ('2002-01-01')),
    PARTITION p_2002 VALUES [('2002-01-01'), ('2003-01-01')),
    PARTITION p_2003 VALUES [('2003-01-01'), ('2004-01-01')),
    PARTITION p_2004 VALUES [('2004-01-01'), ('2005-01-01')),
    PARTITION p_2005 VALUES [('2005-01-01'), ('2006-01-01')),
    PARTITION p_2006 VALUES [('2006-01-01'), ('2007-01-01')),
    PARTITION p_2007 VALUES [('2007-01-01'), ('2008-01-01')),
    PARTITION p_2008 VALUES [('2008-01-01'), ('2009-01-01')),
    PARTITION p_2009 VALUES [('2009-01-01'), ('2010-01-01')),
    PARTITION p_2010 VALUES [('2010-01-01'), ('2011-01-01')),
    PARTITION p_2011 VALUES [('2011-01-01'), ('2012-01-01')),
    PARTITION p_2012 VALUES [('2012-01-01'), ('2013-01-01')),
    PARTITION p_2013 VALUES [('2013-01-01'), ('2014-01-01')),
    PARTITION p_2014 VALUES [('2014-01-01'), ('2015-01-01')),
    PARTITION p_2015 VALUES [('2015-01-01'), ('2016-01-01')),
    PARTITION p_2016 VALUES [('2016-01-01'), ('2017-01-01')),
    PARTITION p_2017 VALUES [('2017-01-01'), ('2018-01-01')),
    PARTITION p_2018 VALUES [('2018-01-01'), ('2019-01-01')),
    PARTITION p_2019 VALUES [('2019-01-01'), ('2020-01-01')),
    PARTITION p_2020 VALUES [('2020-01-01'), ('2021-01-01')),
    PARTITION p_2021 VALUES [('2021-01-01'), ('2022-01-01'))
)
DISTRIBUTED BY HASH(`TRADE_DATE`) BUCKETS 10
PROPERTIES (
  "replication_num" = "1"
);
```


# 22 语法¶
建表时，使用以下语法填充 CREATE-TABLE 时的 partition_info 部分：
AUTO RANGE PARTITION ：
```sql
CREATE TABLE `date_table` (
    `TIME_STAMP` datev2 NOT NULL COMMENT '采集日期'
) ENGINE=OLAP
DUPLICATE KEY(`TIME_STAMP`)
AUTO PARTITION BY RANGE (date_trunc(`TIME_STAMP`, 'month'))
(
)
DISTRIBUTED BY HASH(`TIME_STAMP`) BUCKETS 10
PROPERTIES (
"replication_allocation" = "tag.location.default: 1"
);
```
AUTO LIST PARTITION

```SQL

CREATE TABLE `str_table` (
    `str` varchar not null
) ENGINE=OLAP
DUPLICATE KEY(`str`)
AUTO PARTITION BY LIST (`str`)
(
)
DISTRIBUTED BY HASH(`str`) BUCKETS 10
PROPERTIES (
"replication_allocation" = "tag.location.default: 1"
);

```

3 场景示例¶
在使用场景一节中的示例，在使用 AUTO PARTITION 后，该表 DDL 可以改写为：

```sql
CREATE TABLE `DAILY_TRADE_VALUE`
(
    `TRADE_DATE`              datev2 NOT NULL COMMENT '交易日期',
    `TRADE_ID`                varchar(40) NOT NULL COMMENT '交易编号',
    ......
)
UNIQUE KEY(`TRADE_DATE`, `TRADE_ID`)
AUTO PARTITION BY RANGE (date_trunc(`TRADE_DATE`, 'year'))
(
)
DISTRIBUTED BY HASH(`TRADE_DATE`) BUCKETS 10
PROPERTIES (
  "replication_num" = "1"
);
```
此时新表没有默认分区：

```sql
mysql> show partitions from `DAILY_TRADE_VALUE`;
Empty set (0.12 sec)
```

经过插入数据后再查看，发现该表已经创建了对应的分区：

```sql
mysql> insert into `DAILY_TRADE_VALUE` values ('2012-12-13', 1), ('2008-02-03', 2), ('2014-11-11', 3);
Query OK, 3 rows affected (0.88 sec)

mysql> show partitions from `DAILY_TRADE_VALUE`;
+-------------+-----------------+----------------+---------------------+--------+--------------+--------------------------------------------------------------------------------+-----------------+---------+----------------+---------------+---------------------+---------------------+--------------------------+----------+------------+-------------------------+-----------+
| PartitionId | PartitionName   | VisibleVersion | VisibleVersionTime  | State  | PartitionKey | Range                                                                          | DistributionKey | Buckets | ReplicationNum | StorageMedium | CooldownTime        | RemoteStoragePolicy | LastConsistencyCheckTime | DataSize | IsInMemory | ReplicaAllocation       | IsMutable |
+-------------+-----------------+----------------+---------------------+--------+--------------+--------------------------------------------------------------------------------+-----------------+---------+----------------+---------------+---------------------+---------------------+--------------------------+----------+------------+-------------------------+-----------+
| 180060      | p20080101000000 | 2              | 2023-09-18 21:49:29 | NORMAL | TRADE_DATE   | [types: [DATEV2]; keys: [2008-01-01]; ..types: [DATEV2]; keys: [2009-01-01]; ) | TRADE_DATE      | 10      | 1              | HDD           | 9999-12-31 23:59:59 |                     | NULL                     | 0.000    | false      | tag.location.default: 1 | true      |
| 180039      | p20120101000000 | 2              | 2023-09-18 21:49:29 | NORMAL | TRADE_DATE   | [types: [DATEV2]; keys: [2012-01-01]; ..types: [DATEV2]; keys: [2013-01-01]; ) | TRADE_DATE      | 10      | 1              | HDD           | 9999-12-31 23:59:59 |                     | NULL                     | 0.000    | false      | tag.location.default: 1 | true      |
| 180018      | p20140101000000 | 2              | 2023-09-18 21:49:29 | NORMAL | TRADE_DATE   | [types: [DATEV2]; keys: [2014-01-01]; ..types: [DATEV2]; keys: [2015-01-01]; ) | TRADE_DATE      | 10      | 1              | HDD           | 9999-12-31 23:59:59 |                     | NULL                     | 0.000    | false      | tag.location.default: 1 | true      |
+-------------+-----------------+----------------+---------------------+--------+--------------+--------------------------------------------------------------------------------+-----------------+---------+----------------+---------------+---------------------+---------------------+--------------------------+----------+------------+-------------------------+-----------+
3 rows in set (0.12 sec)
```

经过自动分区功能所创建的 PARTITION ，与手动创建的 PARTITION 具有完全一致的功能性质。

# 3.7.5 手动分桶¶
如果使用了分区，则 DISTRIBUTED ... 语句描述的是数据在各个分区内的划分规则。

如果不使用分区，则描述的是对整个表的数据的划分规则。

也可以对每个分区单独指定分桶方式。

分桶列可以是多列， Aggregate 和 Unique 模型必须为 Key 列， Duplicate 模型可以是 key 列和 value 列。分桶列可以和 Partition 列相同或不同。

分桶列的选择，是在查询吞吐和查询并发之间的一种权衡：

如果选择多个分桶列，则数据分布更均匀。如果一个查询条件不包含所有分桶列的等值条件，那么该查询会触发所有分桶同时扫描，这样查询的吞吐会增加，单个查询的延迟随之降低。这个方式适合大吞吐低并发的查询场景。

如果仅选择一个或少数分桶列，则对应的点查询可以仅触发一个分桶扫描。此时，当多个点查询并发时，这些查询有较大的概率分别触发不同的分桶扫描，各个查询之间的 IO 影响较小（尤其当不同桶分布在不同磁盘上时），所以这种方式适合高并发的点查询场景。


1 Bucket 的数量和数据量的建议¶
一个表的 Tablet 总数量等于（ Partition num * Bucket num ）。

一个表的 Tablet 数量，在不考虑扩容的情况下，推荐略多于整个集群的磁盘数量。

单个 Tablet 的数据量理论上没有上下界，但建议在 1G - 10G 的范围内。

如果单个 Tablet 数据量过小，则数据的聚合效果不佳，且元数据管理压力大。

如果数据量过大，则不利于副本的迁移、补齐，且会增加 Schema Change 或者 Rollup 操作失败重试的代价（这些操作失败重试的粒度是 Tablet ）。

当 Tablet 的数据量原则和数量原则冲突时，建议优先考虑数据量原则。

在建表时，每个分区的 Bucket 数量统一指定。但是在动态增加分区时（ ADD PARTITION ），可以单独指定新分区的 Bucket 数量。可以利用这个功能方便的应对数据缩小或膨胀。

一个 Partition 的 Bucket 数量一旦指定，不可更改。所以在确定 Bucket 数量时，需要预先考虑集群扩容的情况。比如当前只有 3 台 host ，每台 host 有 1 块盘。如果 Bucket 的数量只设置为 3 或更小，那么后期即使再增加机器，也不能提高并发度。

举一些例子：假设在有 10 台 BE ，每台 BE 一块磁盘的情况下。如果一个表总大小为 500MB ，则可以考虑 4-8 个分片。 5GB ： 8-16 个分片。 50GB ： 32 个分片。 500GB ：建议分区，每个分区大小在 50GB 左右，每个分区 16-32 个分片。 5TB ：建议分区，每个分区大小在 50GB 左右，每个分区 16-32 个分片。

Tip

表的数据量可以通过 SHOW DATA 命令查看（该命令的统计结果会有延迟），结果除以副本数，即表的数据量。


# 建议分区，每个分区大小在 50GB 左右，每个分区 16-32 个分片。 

5TB ：建议分区，每个分区大小在 50GB 左右，每个分区 16-32 个分片。
