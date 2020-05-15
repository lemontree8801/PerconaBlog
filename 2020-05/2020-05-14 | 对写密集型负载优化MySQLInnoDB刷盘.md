- [原文链接](https://www.percona.com/blog/2020/05/14/tuning-mysql-innodb-flushing-for-a-write-intensive-workload/)



# 对写密集型负载优化MySQL/InnoDB刷盘
这是详解InnoDB落盘内部原理系列的第三篇，我们将重点介绍调优。（该系列的其他文章为[InnoDB Flushing in Action for Percona Server for MySQL](https://www.percona.com/blog/2020/01/22/innodb-flushing-in-action-for-percona-server-for-mysql/)）与[Give Love to Your SSDs – Reduce innodb_io_capacity_max!](https://www.percona.com/blog/2019/12/18/give-love-to-your-ssds-reduce-innodb_io_capacity_max/)

理解调优过程非常重要，因为我们不想让事情变得更糟或烧毁我们的SSD。我们对每个参数和密切相关的参数进行一节介绍。参数将按MySQL的版本或者是否适用于Percona版MySQL进行分组。如果某一参数在不同版本之间的含义发生变化，则该参数可能会出现多次。可能还有其他参数影响InnoDB处理写密集型负载的方式。希望最重要的参数都已经被涵盖到。

## MySQL8.0.19之前的社区版
### innodb_io_capacity
这个参数的默认值是200。如果您读过我们之前的文章，您就会知道`innodb_io_capacity`是用于空闲刷新和类似后台应用更改缓冲操作的任务的IOPS的数量。自适应刷新与`innodb_io_capacity`无关。在默认值上增加`innodb_io_capacity`理由很少：
- 减少更改缓冲区延迟
- 增加空闲刷新速率（当LSN为常量）
- 增加脏页百分比刷新速率

我们很难找到任何增加空闲刷新速率的原因。此外，不应使用脏页百分比刷新（请参阅下文）。这就只剩下更改缓冲区延迟。如果在执行`show engine innodb status \ G`时，更改缓冲区中经常有大量数据，如下所示
```
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1229, free list len 263690, seg size 263692, 40050265 merges
merged operations:
 insert 98911787, delete mark 40026593, delete 4624943
discarded operations:
 insert 11, delete mark 0, delete 0
```
有1229个未应用的更改，可以考虑提高`innodb_io_capacity`。

### innodb_io_capacity_max
当`innodb_io_capacity`为默认值时，`innodb_io_capacity_max`默认值是2000。这个参数控制InnoDB允许每秒刷新多少页。这大致匹配由InnoDB生成的IOPS。您的硬件应该能够每秒执行刷新那么多页。如果您在MySQL错误日志看到这样的信息：
```
[Note] InnoDB: page_cleaner: 1000ms intended loop took 4460ms. The settings might not be optimal. (flushed=140, during the time.)
```
那么硬件就跟不上了，所以应该降低`innodb_io_capacity_max`的值。理想值应该允许检查点尽可能高，保持在最大检查点的75%以下。检查点越高，数据库需要执行的写操作就越少。与此同时，如果检查点超过75%，您的任务将被阻塞。这需要权衡，写性能与短期阻塞。在所面临的性能痛点和短期风险中作出抉择。如果您已经部署了PMM，那么“MySQL InnoDB Details”Dashboard中的“ InnoDB Checkpoint Age”是非常有用的。下面是sysbech下生成的示例：
![enter image description here](https://www.percona.com/blog/wp-content/uploads/2020/05/cpa.png)

负载从15:02开始，检查点很快增长到20%左右。然后，在15:13，降低了`innodb_io_capacity_max`的值，检查点增长到50%左右。这个增加导致落盘速率降低到一半，同样可以在另一个PMM图"InnoDB Data I / O"看到。
![enter image description here](https://www.percona.com/blog/wp-content/uploads/2020/05/io.png)

Percona版MySQL也公开了所有相关参数：
```
mysql> show global status like 'innodb_ch%';
+---------------------------+------------+
| Variable_name             | Value      |
+---------------------------+------------+
| Innodb_checkpoint_age     |  400343763 |
| Innodb_checkpoint_max_age | 1738750649 |
+---------------------------+------------+
2 rows in set (0.00 sec)
 
mysql> show global variables like 'innodb_adaptive_fl%';
+------------------------------+-------+
| Variable_name                | Value |
+------------------------------+-------+
| innodb_adaptive_flushing     | ON    |
| innodb_adaptive_flushing_lwm | 10    |
+------------------------------+-------+
2 rows in set (0.00 sec)
```
在上面的例子中，检查点大约是最大值的23%，远高于`innodb_adaptive_flushing_lwm`定义的10%。如果检查点是数据库负载下的典型值，并且永远不会升高到更高的值，则可以将`innodb_io_capacity_max`降低一些。相反，如果检查点通常接近甚至超过75%，则`innodb_io_capacity_max`值太小。

### innodb_max_dirty_pages_pct
### innodb_max_dirty_pages_pct_lwm
这些参数在MySQL5.0时代版本中根据缓冲池中脏页的百分比来控制刷新。应通过将`innodb_max_dirty_pages_pct_lwm`设置为0来禁用它。它是5.7中的默认值，但奇怪的是，不是8.0中的默认值。

### innodb_page_cleaners
`innodb_page_cleaners`设置了多个线程，用于扫描缓冲池实例中的脏页并落盘。此外，您不能有比缓冲池实例更多的清理线程。这个参数的默认值为4。

有4个清理线程，InnoDB能够以非常高的速率落盘。实际上，除非您使用Percona版MySQL的并行双写缓冲区功能，否则很可能在清理线程出现瓶颈之前，双写缓冲区就会出现瓶颈。

### innodb_purge_threads
`innodb_purge_threads`定义用于清除操作的线程数。清除操作删除已删除的行，并清除历史记录列表中的undo。这些线程还处理磁盘上undo空间的清理。清除线程默认数量是4。

很难想象一个负载可以让4个清除线程饱和，默认值已足够大。清除线程完成的大部分工作都在内存中。除非有非常大的缓冲池和很高的写负载，否则默认值4就足够了。如果您在“ show engine innodb status”中看到`history length`值非常大，而且没有任何长时间运行的事务，那么您可以考虑增加清除线程的数量。

### innodb_read_io_threads
### innodb_write_io_threads
令人惊讶的是，在异步IO的Linux上，这些线程几乎没有任何关联。例如，我们的一个客户的MySQL服务端有16个读io线程和16个写io线程。正常运行了35天，mysqld进程消耗104小时的CPU时间，读取281TB，并向磁盘写入了48T。数据集主要由压缩的InnoDB表组成。您可能会认为这样的服务端IO线程会很忙，对吗？让我们看看：
```
root@prod:~# ps -To pid,tid,stime,time -p $(pidof mysqld)
   PID TID STIME TIME
 72734  72734 Mar07 00:23:07
 72734  72752 Mar07 00:00:00 -> ibuf (unused)
 72734  77733 Mar07 00:01:16
 72734  77734 Mar07 00:01:51
 72734  77735 Mar07 00:05:46 -> read_io
 72734  77736 Mar07 00:06:04 -> read_io
 72734  77737 Mar07 00:06:06 -> read_io
 … 11 lines removed
 72734  77749 Mar07 00:06:04 -> read_io
 72734  77750 Mar07 00:06:04 -> read_io
 72734  77751 Mar07 00:35:36 -> write_io
 72734  77752 Mar07 00:34:02 -> write_io
 72734  77753 Mar07 00:35:14 -> write_io
 … 11 lines removed
 72734  77766 Mar07 00:35:18 -> write_io
 72734  77767 Mar07 00:34:15 -> write_io
```
我们添加了注释来帮助识别IO线程，并删除了一些行。读IO线程的平均CPU时间为6分钟，而写IO线程的CPU时间略高，为35分钟。这些值非常小，显然，16在两种情况下都太多了。在默认值为4的情况下，读IO线程的CPU时间约为25分钟，写IO线程的CPU时间约为2小时。在34天的正常运行时间中，这些时间不到正常运行时间的1%。

### innodb_lru_scan_depth
`innodb_lru_scan_depth`是一个命名非常糟糕的参数。一个更好的名字是`innodb_free_page_target_per_buffer_pool`。InnoDB试图在每个缓冲池实例中保留一些页面，以加快读取和页创建操作。参数的默认值是1024。

这个参数的危险在于，如果值很高并且缓冲池实例很多，您可能会浪费大量内存。对64个缓冲池实例，默认值为1GB。设置此参数的最佳方法是查看“show engine innodb status \ G”的输出，并对“Free buffers”进行分页查询。以下为示例：
```
mysql> pager grep 'Free buffers'
PAGER set to 'grep 'Free buffers''
mysql> show engine innodb status\G
Free buffers   8187
Free buffers   1024
Free buffers   1023
Free buffers   1022
Free buffers   1020
Free buffers   1024
Free buffers   1025
Free buffers   1025
Free buffers   1024
1 row in set (0.00 sec)
```
第一行是所有缓冲池的总数。正如我们看到的，InnoDB很容易为每个缓冲池实例维护大约1024个空闲页。所有实例中最小值是1020。根据经验，您可以使用当前参数值（这里是1024），减去一定样本数量中观测的最小值，然后将结果乘以4。如果1020是最小值，则为（1024-1020）*4=16。参数的最小允许值为128。我们计算出16，一个小于128的值，所以应使用128。

### innodb_flush_sync
`innodb_flush_sync`允许InnoDB在当检查点超过最大值的94%时忽略`innodb_io_capacity_max`值刷盘。通常这是正确的，因此默认值是“ON”。唯一的例外情况是读SLA非常严格的情况。允许InnoDB在超出`innodb_io_capacity_max`IOPS下刷盘，与读负载竞争资源，并且将丢失读取SLA。

### innodb_log_file_size & innodb_log_files_in_group
`innodb_log_file_size`和`innodb_log_files_in_group`参数决定重做日志循环缓冲区大小。这个话题在我们之前的文章已经详细讨论过了。本质上，重做日志是最近更改的日志。允许的最大总大小是512G减去一个字节。较大的重做日志循环缓冲区允许页在缓冲池中长时间保持脏状态。如果在这段时间内，页收到更多更新，那么写负载实际上就会减少。降低写负载对性能有好处，尤其是数据库接近IO临界点时。一个非常大的重做日志缓冲区会导致崩溃后的恢复花费更多的时间，但是现在这个问题已经不那么严重了。

根据经验，完全使用重做日志的时间应该在一个小时左右。观察`innodb_os_log_written`状态变量随时间的变化，并进行相应的调整。记住，一个小时只是一个普通规律，您的负载可能会有所不同。这是一个非常简单的可调参数，应该是您要调整的第一个参数。

### innodb_adaptive_flushing_lwm
`innodb_adaptive_flushing_lwm`定义了激活自适应刷新的低水位线。低水位线的值表示为检查点超过最大检查点的百分比。当比例低于参数定义的值时，将禁用自适应刷新。默认值是10，最大允许值是70。

更高的水位线的主要好处类似于使用`innodb_log_file_size`更大的值，但是它使数据库更接近最大检查点的边缘。当水位线值很低时，写操作的性能一般会更好，但可能会出现短暂的卡顿。对于生产服务器，增加`innodb_log_file_size`比增加`innodb_adaptive_flushing_lwm`更可取。但是，临时调整，动态增大这个值可以加速逻辑恢复或者使从节点尽快追平主节点。

### innodb_flushing_avg_loops
这个参数控制自适应刷新算法的反应性，它抑制其反应。它影响重做日志的平均消耗量的计算。默认情况下，InnoDB每30秒计算一次平均值，并将结果值与上一个平均值进一步取平均值。这意味着，如果出现事务飙升，自适应算法将反应缓慢，只有与检查点相关部分会立即响应。难以找到修改这个参数的有效案例。

## 8.0.19版本后的MySQL社区版
### innodb_io_capacity
在MySQL8.0.19之前，当InnoDB需要在同步点进行大量刷新时，它会处理大量的页。这种方法的问题是刷盘顺序可能不是最优的，太多页可能来自同一缓冲池实例。

从MySQL8.0.19开始，大量刷新是在`innodb_io_capacity`大小的块完成。块之间没有等待时间，因此与IO容量没有关系。这种方式有助于更好平衡刷新顺序，并降低潜在停顿时间。

### innodb_idle_flush_pct
在MySQL 8.0.19中，空闲刷新率不再由`innodb_io_capacity`直接设置，而是乘以`innodb_idle_flush_pct`表示的百分比。InnoDB插入缓冲线程使用剩余的IO容量。当数据库空闲时，这意味着LSN值不再增加，InnoDB通常首先刷新脏页，然后应用存储在更改缓冲区中的挂起的二级索引更改。如果这不是您想要的行为，请降低该值以向更改缓冲区提供一些IO。

## Percona版的MySQL5.7.x和8.0.x
### innodb_cleaner_lsn_age_factor
Percona版MySQL有不同的自适应刷新算法，`high_checkpoint`,允许有更多的脏页。这个功能由参数`innodb_cleaner_lsn_age_factor`控制。您可以通过将此参数设置为`legacy`恢复MySQL的默认算法。因为您的目标是尽可能多地保留脏页，而不会遇到刷新风暴和停顿，`high_checkpoint`算法可能会帮助您。如果检查点过长，也许`legacy`算法会更好。这个话题更多内容，请参阅我们之前的[博客](https://www.percona.com/blog/2020/01/22/innodb-flushing-in-action-for-percona-server-for-mysql/)。


### innodb_empty_free_list_algorithm
这个参数控制当一个缓冲池实例难以找到可用页时InnoDB的行为。它有两个可能值：`legacy`和`backoff`。这个仅在您` "show engine innodb status \ G"`的 `"Free buffers"`接近为0时有意义。在这个场景下，LRU管理线程可能会破坏空闲列表的互斥锁，并引起与其他线程的争用。当设置为`backoff`时，线程将在找不到可用页时休眠一段时间以降低争用。默认值为`backoff`，通常适用于大多数场景。

## 总结
还有其他影响InnoDB写入的参数，但是这些是最重要的部分。作为一个基本规则，不要过度优化您的数据库服务，一次性仅调整一个参数。

这是我们对影响刷盘的InnoDB参数的理解。这些发现是我们在Percona对各种各样的用户案例总结的。我们还花了很多时间阅读代码来理解InnoDB内部移动部分。像往常一样，我们开放了评论，但是如果可能的话，请引用代码或者可复现的用例来支持或者反对我们的观点。
