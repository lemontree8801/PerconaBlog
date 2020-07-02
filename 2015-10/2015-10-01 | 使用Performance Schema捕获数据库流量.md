- [原文链接](https://www.percona.com/blog/2015/10/01/capture-database-traffic-using-performance-schema/)

# 使用Performance Schema捕获数据库流量
捕获数据是进行SQL分析的关键部分，甚至仅是了解数据库内部的情况。

有几种已知方法来实现。例如：
- 启用通用日志
- 设置`long_query_time=0`后，启用慢日志
- 使用[TCPDUMP](https://www.percona.com/blog/2008/11/07/poor-mans-query-logging/)从网络流量捕获搭配MySQL的数据包
- 使用`pt-query-digest`的`--processlist`参数

但是，这些方法可能会增加大量的开销，甚至可能对性能产生负面影响，例如：
- 使用`SHOW PROCESSLIST`需要一个[Mutex](https://dev.mysql.com/doc/refman/5.6/en/show-processlist.html)
- 记录所有查询会[降低MySQL性能](https://www.percona.com/blog/2009/02/10/impact-of-logging-on-mysql%E2%80%99s-performance/)，尤其是在CPU密集型场景
- 日志切割可能[会有问题](https://www.percona.com/blog/2012/09/20/a-prototype-lower-impact-slow-query-log/)

现在，有时候您只需要查看流量高峰情况。这种情况下，使用`pt-query-digest`并添加`--processlist`参数可能是收集一些流量数据更快和最简单的方法。它不需要对服务端的配置修改，也不需要对文件进行关键处理。它甚至不需要登陆服务端，只需要具有`show full processlist`的权限用户即可。但是，您必须考虑到使用`SHOW PROCESSLIST`命令会错过很多查询，并且时序分别率非常差，并还有一些其他的问题（例如processlist Mutex）。

有什么替代方法吗？使用**Performance Schema**。尤其是：
**`events_statements_*`**
和
**`threads`**
表。

首先，我们必须确保启用了相应的消费者：
```
mysql> select * from setup_consumers where name like 'events%statement%' and enabled = 'yes';
+--------------------------------+---------+
| NAME                           | ENABLED |
+--------------------------------+---------+
| events_statements_current      | YES     |
| events_statements_history      | YES     |
| events_statements_history_long | YES     |
+--------------------------------+---------+
3 rows in set (0.00 sec)
```

此外，对于要收集的语句统计信息，仅启用用于单个语句类型的
**`statement/sql/*`**
是不够的。还需要启用
**`statement/abstract/*`**
。这通常不应该是一个问题，因为所有的语句消费者默认情况下都是启用的。

如果您`setup_consumer`表中看不到
**`event_statements_*`**
消费者，您可能运行的是5.6.3之前的MySQL版本。在此版本之前，
**`event_statements_*`**
表并不存在。正如之前一篇[博客](https://www.percona.com/blog/2014/02/11/performance_schema-vs-slow-query-log/)已经指出的那样，不应再继续使用MySQL5.6。

在继续之前，要注意捕获数据的最重要条件是：
>**语句已经提交。**

如果语句还在执行中，它不能成为被收集的流量的一部分。对于那些想知道MySQL内部在运行什么内容的人，在Sys Schema中有一个详细的非阻塞的processlist视图可以替换[INFORMATION_SCHEMA.|SHOW FULL PROCESSLIST]（在MySQL5.7版中是默认）

我们从三个可用表中获取数据：`events_statements_current`， `events_statements_history` 或者 `events_statements_history_long`。

- 第一个选择是使用`events_statements_current`表，其中包含当前语句事件。因为我们只想获得已经结束的语句，所以需要添加**END_EVENT_ID IS NOT NULL**条件。该列在事件开始时设置为NULL，当事件结束后，更新为线程当前事件号，但在测试时，发现有太多SQL丢失。这可能是因为在迭代时，关联的线程被从`threads`表中删除，或者只是因为更新**END_EVENT_ID**和从表中删除数据之间的时间太短。这个选择被放弃。

- 第二个选择是`events_statements_history`表包含每个线程最近的语句事件，直到语句已经结束，语句事件不会添加到`events_statements_history`表中。了解事件是否仍在运行，使用这个表不需要添加额外条件。此外，当表已经满，添加新事件时，旧事件将被删除。

	这意味着这个表大小是固定的。您可以通过修改参数`performance_schema_events_statements_history_size`更改表的大小。在我使用的版本中（Percona 5.6.25-73.1），表的默认大小为`autosized`（-1），每个线程可以有10行。例如：如果运行5个线程，表将会有50行。

	由于它的大小是固定的，所以在迭代时可能会丢失一些事件。

- 第三个选项是`events_statements_history_long`表是`events_statements_history`表的扩展版本。取决于MySQL的版本，默认可以存放10000行或者`autosized`，同样可以通过修改参数`performance_schema_events_statements_history_long_size`修改。

	这个表的一个主要缺点是“当线程结束时，它的数据将从表中删除”。所以它不是历史数据。它会记录最旧的线程，而较旧的事件依然存在。

原则上使用第三个选择：`events_statements_history_long`表。我写了一个小[脚本](https://github.com/nethalo/mysql-tools/blob/master/get_history_long.sh)来收集event_id范围内每个线程上所有事件的迭代。范围的概念是避免重复捕获同一事件。事实证明，查询这个表数据非常慢，大约在0.53秒到1.96秒之间。它侵入性很强。

仅剩下第二个选择：`events_statements_history`表。

由于目标是以慢日志格式的方式捕获数据，因此需要从threads表获取额外信息，该表为每个线程记录一行。需要记住的最重要的事情是：访问线程不需要mutex，对服务端性能影响最小。

结合这两个表，我们可以获得足够的信息来模拟一种非常全面的慢日志格式。我们只需要合适的查询：
```
SELECT
CONCAT_WS(
'','# Time: ', date_format(CURDATE(),'%y%m%d'),' ',TIME_FORMAT(NOW(6),'%H:%i:%s.%f'),'\n'
,'# User@Host: ',t.PROCESSLIST_USER,'[',t.PROCESSLIST_USER,'] @ ',PROCESSLIST_HOST,' []  Id: ',t.PROCESSLIST_ID,'\n'
,'# Schema: ',CURRENT_SCHEMA,'  Last_errno: ',MYSQL_ERRNO,'  ','\n'
,'# Query_time: ',ROUND(s.TIMER_WAIT / 1000000000000, 6),' Lock_time: ',ROUND(s.LOCK_TIME / 1000000000000, 6),'  Rows_sent: ',ROWS_SENT,'  Rows_examined: ',ROWS_EXAMINED,'  Rows_affected: ',ROWS_AFFECTED,'\n'
,'# Tmp_tables: ',CREATED_TMP_TABLES,'  Tmp_disk_tables: ',CREATED_TMP_DISK_TABLES,'  ','\n'
,'# Full_scan: ',IF(SELECT_SCAN=0,'No','Yes'),'  Full_join: ',IF(SELECT_FULL_JOIN=0,'No','Yes'),'  Tmp_table: ',IF(CREATED_TMP_TABLES=0,'No','Yes'),'  Tmp_table_on_disk: ',IF(CREATED_TMP_DISK_TABLES=0,'No','Yes'),'\n'
, t.PROCESSLIST_INFO,';')
FROM performance_schema.events_statements_history s
JOIN performance_schema.threads t using(thread_id)
WHERE
t.TYPE = 'FOREGROUND'
AND t.PROCESSLIST_INFO IS NOT NULL
AND t.PROCESSLIST_ID != connection_id()
ORDER BY t.PROCESSLIST_TIME desc;
```

这个查询的思想是获得与使用`log_slow_filter`参数获取到的慢日志格式尽可能相似。

其他条件包括：

- **t.TYPE='FOREGROUND'**：`threads`表提供了关于后台线程的信息，我们不打算对其进行分析。用户连接线程是前台线程。
- **t.PROCESSLIST_INFO IS NOT NULL**：如果线程没有执行任何语句，这个字段为NULL。
- **t.PROCESSLIST_ID != connection_id()**：忽略自身（这个查询）。

SQL输出结果与慢日志输出类似：
```
# Time: 150928 18:13:59.364770
# User@Host: root[root] @ localhost []  Id: 58918
# Schema: test  Last_errno: 0
# Query_time: 0.000112 Lock_time: 0.000031  Rows_sent: 1  Rows_examined: 1  Rows_affected: 0
# Tmp_tables: 0  Tmp_disk_tables: 0
# Full_scan: No  Full_join: No  Tmp_table: No  Tmp_table_on_disk: No
INSERT INTO sbtest1 (id, k, c, pad) VALUES (498331, 500002, '94319277193-32425777628-16873832222-63349719430-81491567472-95609279824-62816435936-35587466264-28928538387-05758919296'
, '21087155048-49626128242-69710162312-37985583633-69136889432');
```

这个文件可以与`pt-query-digest`一起用于聚合类似的查询，就像它是常规的慢日志输出一样。我进行了一个小测试，其中包括：

- 使用`sysbench`生成流量。这是使用`sysbench`命令：
```
sysbench --test=/usr/share/doc/sysbench/tests/db/oltp.lua --mysql-host=localhost --mysql-port=3306 --mysql-user=root --mysql-password= --mysql-db=test --mysql-table-engine=innodb --oltp-test-mode=complex --oltp-read-only=off --oltp-reconnect=on --oltp-table-size=1000000 --max-requests=100000000 --num-threads=4 --report-interval=1 --report-checkpoints=10 --tx-rate=0 run
```
- 使用慢日志，设置`long_query_time=0`，然后捕获数据。
- 使用`pt-query-digest --processlist`捕获数据。
- 使用`Performance Schema`捕获数据。
- 用`pt-query-digest`分析这3个文件。

结果为：
- 慢日志：
```
# Profile
# Rank Query ID           Response time Calls  R/Call V/M   Item
# ==== ================== ============= ====== ====== ===== ==============
#    1 0x813031B8BBC3B329 47.7743 18.4%  15319 0.0031  0.01 COMMIT
#    2 0x737F39F04B198EF6 39.4276 15.2%  15320 0.0026  0.00 SELECT sbtest?
#    3 0x558CAEF5F387E929 37.8536 14.6% 153220 0.0002  0.00 SELECT sbtest?
#    4 0x84D1DEE77FA8D4C3 30.1610 11.6%  15321 0.0020  0.00 SELECT sbtest?
#    5 0x6EEB1BFDCCF4EBCD 24.4468  9.4%  15322 0.0016  0.00 SELECT sbtest?
#    6 0x3821AE1F716D5205 22.4813  8.7%  15322 0.0015  0.00 SELECT sbtest?
#    7 0x9270EE4497475EB8 18.9363  7.3%   3021 0.0063  0.00 SELECT performance_schema.events_statements_history performance_schema.threads
#    8 0xD30AD7E3079ABCE7 12.8770  5.0%  15320 0.0008  0.01 UPDATE sbtest?
#    9 0xE96B374065B13356  8.4475  3.3%  15319 0.0006  0.00 UPDATE sbtest?
#   10 0xEAB8A8A8BEEFF705  8.0984  3.1%  15319 0.0005  0.00 DELETE sbtest?
# MISC 0xMISC              8.5077  3.3%  42229 0.0002   0.0 <10 ITEMS>
```

- `pt-query-digest --processlist`
```
# Profile
# Rank Query ID           Response time Calls R/Call V/M   Item
# ==== ================== ============= ===== ====== ===== ===============
#    1 0x737F39F04B198EF6 53.4780 16.7%  3676 0.0145  0.20 SELECT sbtest?
#    2 0x813031B8BBC3B329 50.7843 15.9%  3577 0.0142  0.10 COMMIT
#    3 0x558CAEF5F387E929 50.7241 15.8%  4024 0.0126  0.08 SELECT sbtest?
#    4 0x84D1DEE77FA8D4C3 35.8314 11.2%  2753 0.0130  0.11 SELECT sbtest?
#    5 0x6EEB1BFDCCF4EBCD 32.3391 10.1%  2196 0.0147  0.21 SELECT sbtest?
#    6 0x3821AE1F716D5205 28.1566  8.8%  2013 0.0140  0.17 SELECT sbtest?
#    7 0x9270EE4497475EB8 22.1537  6.9%  1381 0.0160  0.22 SELECT performance_schema.events_statements_history performance_schema.threads
#    8 0xD30AD7E3079ABCE7 15.4540  4.8%  1303 0.0119  0.00 UPDATE sbtest?
#    9 0xE96B374065B13356 11.3250  3.5%   885 0.0128  0.09 UPDATE sbtest?
#   10 0xEAB8A8A8BEEFF705 10.2592  3.2%   792 0.0130  0.09 DELETE sbtest?
# MISC 0xMISC              9.7642  3.0%   821 0.0119   0.0 <3 ITEMS>
```

- `Performance Schema`
```
# Profile
# Rank Query ID           Response time Calls R/Call V/M   Item
# ==== ================== ============= ===== ====== ===== ==============
#    1 0x813031B8BBC3B329 14.6698 24.8% 12380 0.0012  0.00 COMMIT
#    2 0x558CAEF5F387E929 12.0447 20.4% 10280 0.0012  0.00 SELECT sbtest?
#    3 0x737F39F04B198EF6  7.9803 13.5% 10280 0.0008  0.00 SELECT sbtest?
#    4 0x3821AE1F716D5205  4.6945  7.9%  5520 0.0009  0.00 SELECT sbtest?
#    5 0x84D1DEE77FA8D4C3  4.6906  7.9%  7350 0.0006  0.00 SELECT sbtest?
#    6 0x6EEB1BFDCCF4EBCD  4.1018  6.9%  6310 0.0007  0.00 SELECT sbtest?
#    7 0xD30AD7E3079ABCE7  3.7983  6.4%  3710 0.0010  0.00 UPDATE sbtest?
#    8 0xE96B374065B13356  2.3878  4.0%  2460 0.0010  0.00 UPDATE sbtest?
#    9 0xEAB8A8A8BEEFF705  2.2231  3.8%  2220 0.0010  0.00 DELETE sbtest?
# MISC 0xMISC              2.4961  4.2%  2460 0.0010   0.0 <7 ITEMS>
```
Performance Schema的数据比使用常规`SHOW FULL PROCESSLIST`捕获的数据更接近慢日志数据，但它仍然很不准确。这是一种快速、简单的获取流量的替代方法，因此您可能不得不接受这种权衡。

## 总结
捕获流量是需要权衡的，但是如果您愿意牺牲准确性，那么使用Performance Schema可以在对服务端性能影响最小的情况下完成。从MySQL5.6开始，P_S是默认启用的，假如您已经使用5.6，您可能有P_S启用的开销。如果您在生产中已经启用P_S，不要害怕使用它。里面已经有很多数据了。
