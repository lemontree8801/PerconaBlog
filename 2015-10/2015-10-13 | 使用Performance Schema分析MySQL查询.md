- [原文链接](https://www.percona.com/blog/2015/10/13/mysql-query-digest-with-performance-schema/)


# 使用Performance Schema分析MySQL查询
分析查询是实现高性能的最佳方法。这也可能是DBA日常工作最常重复的事。对我们中的大多数而言会选择`pt-query-digest`这个工具，它是分析慢查询的最佳工具之一。

为什么不使用`pt-query-digest`？有时候获得慢日志比较困难，例如RDS实例或者当您的数据库是DBaaS的一部分（这在某些组织中是一种常见的做法）。

在这种情况下，最好有一个替代方案。在这种情况下，可以选择使用`Performance Schema`。我们之前已经讨论了关于[`events_statements_* tables`](https://www.percona.com/blog/2015/10/01/capture-database-traffic-using-performance-schema/)  ，现在是使用`events_statements_summary_by_digest`表的时候了。在这个表中，每一行汇总了指定schema/SQL摘要的事件（注意，在MySQL5.6.9之前，没有`SCHEMA_NAME`列，只是基于SQL摘要进行分组）。

为了让MySQL可以在汇总表中聚合信息，必须验证已经启用了`statement_digest`的消费者。

获取数据最直接的方法是查询表，如下所示：
```
SELECT
SCHEMA_NAME,
digest,
digest_text,
round(sum_timer_wait/ 1000000000000, 6),
count_star
FROM performance_schema.events_statements_summary_by_digest
ORDER BY sum_timer_wait DESC LIMIT 10;
```

 这显示了服务中SQL语句的数量和频率。非常简单。但也有一些注意事项：
 - **语句将被格式化为摘要文本。**
 你将不是看到`SELECT age FROM population WHERE id BETWEEN 153 AND 153+69`，您将看到的是SQL摘要：`SELECT age FROM population WHERE id BETWEEN ? AND ? + ?`
 - `events_statements_summary_by_digest`的最大行数是有限制的（默认为200，从MySQL5.6.5开始可以使用参数`performance_schema_digests_size`修改）。As a consequence, when the table is full, statement digest values that have no already existing row will be added to a special “catch-all” row with DIGEST = NULL. In plain English: you won’t have meaningful info for those statements.

要解决第一个问题，我们可以使用**events_statements_history**表来获得所有的摘要查询。我选择不使用**events_statements_currents**，是因为行在表中的生命周期很短。对于history表，在相同的时间内有更多的机会得到更多的查询。

当使用`pt-query-digest`时，第一步总是收集具有一定量的数据，通常是读取慢日志，然后进行处理。使用Performance Schema，收集具有代表性的完整查询，这样我们就可以为每条聚合语句提供示例。

为解决第二个问题，我们只需要`TRUNCATE events_statements_summary_by_digest`。这样，汇总表就会零开始。

由于Performance Schema在MySQL支持的所有平台都可用，所以我选择在Amazon RDS MySQL实例上进行测试。我唯一不喜欢的是在**RDS上P_S上默认是禁用的**，启用它需要重新启动实例。除此之外，一切都与常规实例相同。

步骤是：
1. 启用`events_statements_history`消费者
2. 创建内存表来存储数据
3. 清空表重新开始
4. 创建MySQL EVENT，来填充表
5. 事件结束后，获取查询摘要。

表结构如下：
```
CREATE TABLE IF NOT EXISTS percona.digest_seen
(schema_name varchar(64) DEFAULT NULL,
digest varchar(32) DEFAULT NULL,
sql_text varchar(1024) DEFAULT NULL,
PRIMARY KEY USING BTREE (schema_name,digest)) engine=memory;
```

`events_statements_history`表的原始`SQL_TEXT`字段被定义为`longtext`字段，但是除非使用Percona版本（5.5+），否则您不能在内存表上使用`longtext`字段。这在Percona版本中是可能的，因为改进的内存存储引擎允许在内存存储引擎上使用`Blob`和`Text`字段。解决方法是将该字段定义为varchar 1024。为什么是1024？这是该表的另一个需求：`SQL_TEXT`固定在1024字符。只有在MySQL5.7.6之后，才能通过在服务启动时改变`performance_schema_max_sql_text_length`系统参数来修改显示的最大字节数。

另外，由于我们将在RDS上使用`EVENTS`，因此必须要将参数`event_scheduler`设置为`ON`。幸运的是，它是一个动态参数，因此在修改参数后不需要重启实例。如果使用非RDS，则可以执行`SET GLOBAL event_scheduler = ON;`

下面是完整步骤：
```
UPDATE performance_schema.setup_consumers SET ENABLED = 'YES' WHERE NAME = 'events_statements_history';
SET SESSION max_heap_table_size = 1024*1024;
CREATE DATABASE IF NOT EXISTS percona;
USE percona;
CREATE TABLE IF NOT EXISTS percona.digest_seen (schema_name varchar(64) DEFAULT NULL, digest varchar(32) DEFAULT NULL, sql_text varchar(24) DEFAULT NULL, PRIMARY KEY USING BTREE (schema_name,digest)) engine=memory;

TRUNCATE TABLE percona.digest_seen;
TRUNCATE performance_schema.events_statements_summary_by_digest;
TRUNCATE performance_schema.events_statements_history;

CREATE EVENT IF NOT EXISTS getDigest
ON SCHEDULE EVERY 1 SECOND
ENDS CURRENT_TIMESTAMP + INTERVAL 5 MINUTE ON COMPLETION NOT PRESERVE
DO
INSERT IGNORE INTO percona.digest_seen SELECT CURRENT_SCHEMA, DIGEST, SQL_TEXT FROM performance_schema.events_statements_history WHERE DIGEST IS NOT NULL GROUP BY current_schema, digest LIMIT 50;
```

`Event`被定义为立即运行，每秒运行一次，持续五分钟。事件完成后，它将被删除。

当事件完成后，我们就可以获得查询摘要了。只需要执行此查询：
```
SELECT
s.SCHEMA_NAME,
s.SQL_TEXT,
ROUND(d.SUM_TIMER_WAIT / 1000000000000, 6) as EXECUTION_TIME,
ROUND(d.AVG_TIMER_WAIT / 1000000000000, 6) as AVERAGE_TIME,
COUNT_STAR
FROM performance_schema.events_statements_summary_by_digest d
LEFT JOIN percona.digest_seen s USING (digest)
WHERE s.SCHEMA_NAME IS NOT NULL
GROUP BY s.digest
ORDER BY EXECUTION_TIME DESC LIMIT 10;
```

排序顺序与`pt-query-digest`的顺序类似，但可以是您想要的任何顺序。

输出为：
```
+-------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------+--------------+------------+
| SCHEMA_NAME | SQL_TEXT                                                                                                                                                                  | EXECUTION_TIME | AVERAGE_TIME | COUNT_STAR |
+-------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------+--------------+------------+
| percona     | UPDATE population SET age=age+1 WHERE id=148                                                                                                                              |  202304.145758 |     1.949487 |     103773 |
| percona     | SELECT age FROM population WHERE id BETWEEN 153 AND 153+69                                                                                                                |     488.572609 |     0.000176 |    2771352 |
| percona     | SELECT sex,age,estimate2012 FROM population WHERE id BETWEEN 174 AND 174+69 ORDER BY sex                                                                                  |     108.841575 |     0.000236 |     461412 |
| percona     | SELECT census2010 FROM population WHERE id=153                                                                                                                            |      62.742239 |     0.000090 |     693526 |
| percona     | SELECT SUM(estimate2014) FROM population WHERE id BETWEEN 154 AND 154+69                                                                                                  |      44.940020 |     0.000195 |     230810 |
| percona     | SELECT DISTINCT base2010 FROM population WHERE id BETWEEN 154 AND 154+69 ORDER BY base2010                                                                                |      33.909593 |     0.000294 |     115338 |
| percona     | UPDATE population SET estimate2011='52906609184-39278192019-93190587310-78276160274-48170779146-66415569224-40310027367-70054020251-87998206812-01032761541' WHERE id=154 |       8.231353 |     0.000303 |      27210 |
| percona     | COMMIT                                                                                                                                                                    |       2.630153 |     0.002900 |        907 |
| percona     | BEGIN                                                                                                                                                                     |       0.705435 |     0.000031 |      23127 |
|             | SELECT 1                                                                                                                                                                  |       0.422626 |     0.000102 |       4155 |
+-------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------+--------------+------------+
10 rows in set (0.10 sec)
```

最后，您可以做这些清理工作：
```
DROP event IF EXISTS getDigest;
DROP TABLE IF EXISTS percona.digest_seen;
SET SESSION max_heap_table_size = @@max_heap_table_size;
UPDATE performance_schema.setup_consumers SET ENABLED = 'NO' WHERE NAME = 'events_statements_history';
```

**总结：** Performance Schema已经为您完成了查询摘要。问题是如何以适合您的需求方式访问数据。
