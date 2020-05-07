- [原文链接](https://www.percona.com/blog/2020/05/07/prepare-mysql-for-a-safe-shutdown/)


# MySQL安全关闭前做的准备
我们启动MySQL只收到了一些可怕的、意想不到的错误。或者我们在完成关机之前已经等了好几个小时，而它只是夯在了某个任务上。

无论是进行维护还是升级，我们都可以采取一些步骤来为MySQL的安全做准备，希望能够清除错误日志。

注意：以下步骤假设是将要停机一个异步复制的从节点。
## 1.停止复制
在某些（罕见的）情况下，一个从节点可能尝试在错误的位点启动。为了最小化这种风险，首先停止IO线程，这样它就不会接收到新的事件。
```
STOP SLAVE IO_THREAD;
```
等待SQL线程应用完所有事件，也停掉它。
```
SHOW SLAVE STATUS\G
STOP SLAVE SQL_THREAD;
```
这将使两个复制线程处于一致的位置，因此中继日志只包含已执行的事件，而`relay_log_info_repository`位置是最新的。我们会尽快清除日志，以防万一。

对于多源复制，请确保在停止复制之前填补空缺。
```
STOP SLAVE;
START SLAVE UNTIL SQL_AFTER_MTS_GAPS;
SHOW SLAVE STATUS\G # Make sure the SQL_thread has stopped before proceeding.
STOP SLAVE;
```

## 2.提交，回滚或者杀掉长时间运行的事务
在一分钟内可以发生很多事，InnoDB必须在关机期间回滚未提交的事务。这非常昂贵并可能需要很长时间。事务应该在MySQL运行时处理。

以下查询将展示运行时间大于60秒的事务，并返回了一些元数据。利用这些细节进一步研究并决定如何进行。
```
mysql> SELECT trx_id, trx_started, (NOW() - trx_started) trx_duration_seconds, id processlist_id, user, IF(LEFT(HOST, (LOCATE(':', host) - 1)) = '', host, LEFT(HOST, (LOCATE(':', host) - 1))) host, command, time, REPLACE(SUBSTRING(info,1,25),'\n','') info_25 FROM information_schema.innodb_trx JOIN information_schema.processlist ON innodb_trx.trx_mysql_thread_id = processlist.id WHERE (NOW() - trx_started) > 60 ORDER BY trx_started;
+--------+---------------------+----------------------+----------------+------+-----------+---------+------+---------------------------+
| trx_id | trx_started         | trx_duration_seconds | processlist_id | user | host      | command | time | info_25                   |
+--------+---------------------+----------------------+----------------+------+-----------+---------+------+---------------------------+
| 511239 | 2020-04-22 16:52:23 |                 2754 |           3515 | dba  | localhost | Sleep   | 1101 | NULL                      |
| 511240 | 2020-04-22 16:53:44 |                   74 |           3553 | root | localhost | Query   |   38 | update t1 set name="test" |
+--------+---------------------+----------------------+----------------+------+-----------+---------+------+---------------------------+
2 rows in set (0.00 sec)
```
注意：这个查询还会展示XA PREPARED事务，这些事务可能不应人工提交，回滚或者杀掉。

## 3.清理porcesslist
MySQL将关闭并终止所有连接。因此本着团队合作的精神，我们可以伸出援手！

使用`pt-kill`检查并杀掉活跃和休眠连接。在我的示例中，我将忽略**percona**和**orchestrator**用户。
```
pt-kill --host="localhost" --victims="all" --interval=10 --ignore-user="percona|orchestrator" --busy-time=1 --idle-time=1 --print [--kill]
```
这将处理服务端上占用资源的所有长时间休眠的连接和活跃连接。

## 4.配置InnoDB，最快时间落盘
在关闭MySQL进行升级时，您会经常看到类似的建议。理想情况下，它每次都被执行，以便为关机做好安全准备。
```
SET GLOBAL innodb_fast_shutdown=0;
SET GLOBAL innodb_max_dirty_pages_pct=0;
SET GLOBAL innodb_change_buffering='none';
```
现在监控脏页的数量。期望结果是0，但是如果MySQL上仍然有活动，则并非总是可能的。如果它看来没有增加（计数器并没有持续下降），就认为它是好的，进入下一步。
```
SHOW GLOBAL STATUS LIKE '%dirty%';
```
减少`innodb_max_dirty_pages_pct`将增加准备步骤的时间，因为您要等待缓冲池脏页计数尽可能接近于0。但是，这减少了关闭时间，因为页已经都落盘了。

当您等待完整的undo log清理和change buffer 合并时，禁用`innodb_fast_shutdown`可能会使实际关闭时间增加数分钟或数小时，所以我们也停止了`innodb_change_buffering`。如果您使用PMM，您可以在`InnoDB Change Buffer`图中查看大小和活跃情况。

## 5.Dump 缓冲池
这个主机很有可能再次加入集群。Dump和加载缓冲池内容将大大减少启动后的预热时间。

在MySQL运行时，dump缓冲池。
```
SET GLOBAL innodb_buffer_pool_dump_pct=75;
SET GLOBAL innodb_buffer_pool_dump_now=ON;
```
监控`status`，等待其完成。
```
mysql> SHOW STATUS LIKE 'Innodb_buffer_pool_dump_status';
+--------------------------------+--------------------------------------------------+
| Variable_name                  | Value                                            |
+--------------------------------+--------------------------------------------------+
| Innodb_buffer_pool_dump_status | Buffer pool(s) dump completed at 200429 14:04:47 |
+--------------------------------+--------------------------------------------------+
1 row in set (0.01 sec)
```
要确保缓冲池将被加载，请检查配置文件中的`innodb_buffer_pool_load_at_startup`是否被禁用（这是一个默认为启用的全局只读参数）。

## 6.刷新日志
正如第一步中所提到的，我们将刷新日志。
```
FLUSH LOGS;
```
MySQL为安全关闭已经准备完成！

## 总结
大多数时，DBA只需要发出`stop`命令，MySQL就会正常关闭（这是事实）。但是，有一次不太顺利...发生了什么？采取类似这些步骤，以确保安全关闭。

不要等待“一次”的到来。现在创建一个安全的步骤，并为额外的步骤保持耐心！
