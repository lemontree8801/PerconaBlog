- [原文链接](https://www.percona.com/blog/2016/05/03/best-practices-for-configuring-optimal-mysql-memory-usage/)


# 配置最佳MySQL内存使用的最佳实践
在这篇博文中，我们将讨论配置最佳MySQL内存使用的一些最佳实践。

正确配置可用内存资源是使MySQL获得最佳性能和稳定性的最重要的事情之一。从MySQL5.7开始，默认配置使用非常有限的内存——保留默认值是最糟糕的事情之一。但是不正确地配置它会导致更糟糕的性能（甚至崩溃）。

配置MySQL内存使用的第一规则是，**永远不要因MySQL导致操作系统使用swap**。即使是很小的swap活动也会极大地降低MySQL性能。注意这里的关键字“活动”。在swap文件中保留一些使用过的空间没有关系，因为当MySQL运行时，操作系统中可能有一些部分是未使用的，所以最好将它们交换出去。您不希望在操作期间频繁地进行swap，这在`vmstat`输出的"si"和"so"列中很容易看到。

![enter image description here](https://www.percona.com/blog/wp-content/uploads/2016/05/No-Significant-Swapping.png)

举例：没有大规模Swap

![enter image description here](https://www.percona.com/blog/wp-content/uploads/2016/05/Heavy-Swapping.png)

举例：大规模Swap

如果您正在使用PMM，您还可以查看`System Overview`Dashboard中的`Swap Activity`图。
![enter image description here](https://www.percona.com/blog/wp-content/uploads/2016/05/Swap-activity.png)

如果峰值超过1MB/秒，或者频繁地swap，那么可能需要重新检查内存配置。

MySQL内存分配很复杂。有全局缓冲区、每个连接缓冲区（取决于负载）和一些不受控制的内存分配（如，内部存储过程），所有的这些都导致难以计算MySQL将真正为您的负载使用多少内存。最好通过查看MySQL使用的虚拟内存大小（VSZ）进行检查。您可以使用`top`或者执行`ps aux | grep mysqld`。

```
mysql     3939 30.3 53.4 11635184 8748364 ?    Sl   Apr08 9106:41 /usr/sbin/mysqld
```

第5列显示了虚拟内存使用情况（大约11G）。

注意，VSZ可能会随时间变化。好的方法是在您的监控系统中展示它并设置告警，以便在它达到指定的阈值时通知您。**不允许`mysqld`进程的VSZ超出系统内存的90%**（如果系统上运行的不仅是MySQL，那么就应更少）。

为了安全起见，最好一开始就保守地设置全局缓冲区和每个连接缓冲区，然后按需增加它们。很多设置都可以在线设置，包括从MySQL5.7中的`innodb_buffer_pool_size`。

## MySQL需要分配多少内存
如何决定分配多少内存给MySQL？在大多数情况下，不应该将超过90%的物理内存分配给MySQL，因为需要为操作系统预留一些空间，比如缓存二进制文件、临时排序文件等。

有些情况下，MySQL使用的内存应该大大低于90%：

- 如果在同一服务器上有其他重要的进程在运行，亦或是一直运行还是定期运行。如果您的定时任务中有大量的批处理作业，这需要大量的内存，您需要考虑这一点。
- 如果您想对某些存储引擎使用操作系统缓存。对于InnoDB，我们建议在大多数情况下使用`innodb_flush_method=O_DIRECT`，不使用操作系统文件缓存。在某些情况下，InnoDB使用缓冲IO是合情合理的。如果您还在使用MyISAM，您需要操作系统缓存您的表“数据”部分。
- 如果您的负载有很大的需求，操作系统的缓存——MyISAM磁盘临时表、排序文件、MySQL需要创建的一些其他的临时文件用于缓存，以获得最佳性能。

一旦知道您希望MySQL进程整体拥有多少内存，您就需要考虑MySQL内部应该把内存用在什么地方。MySQL中内存使用的一部分是与负载相关的——如果同时有许多链接处于活动状态，并且使用大量内存进行排序或者临时表，那么您可能需要大量内存（特别是在启用了Performance_Schema的情况下）。在其他情况下，这个内存量是最小的。为此您通常需要**1到10GB**的空间。

您需要考虑的另一件事是**内存碎片**。根据您在使用的内存分配库（glibc、TCMalloc、jemalloc等），操作系统的设置，比如透明大页和负载可能会显示内存使用量随时间增长（直到达到某个稳定状态）。内存碎片可能占用额外的10%的内存或者更多。

最后，让我们考虑各种全局缓冲区和缓存。在典型的情况下，您主要只需要担心`innodb_buffer_pool_size`。您可能还需要考虑`key_buffer_size`、`query_cache_size`以及`table_cache`和`table_open_cache`。它们负责全局内存分配，即使它们不是按字节计算的。Performance_Schema还可能占用大量内存，特别是在系统中有大量链接或表的情况下。

在指定缓冲区和内存大小时，应该要理解指定的缓存的具体含义。对于`innodb_buffer_pool_size`，请记住，还有另外5-10%的内存会分配给其他数据结构——如果使用压缩或将`innodb_page_size`设置为小于16K，那么这个数字会更大。

对于大容量内存的系统，数据库缓存是最大的内存消耗者，并且您为其分配大部分内存。当您向系统添加额外的内存条时，通常是增加数据库缓存的大小。

让我们对一个具体的例子做计算。假设您有一个拥有16GB内存的系统（物理机或者虚拟机）。我们只在这个系统上运行MySQL，使用InnoDB存储引擎，`innodb_flush_method=O_DIRECT`，所以我们可以为MySQL分配90%的内存，即14.4GB。对于负载，我们假设连接处理和其他基于MySQL连接的开销将占用1GB（还剩余13.4GB）。各种全局缓冲区（`innodb_log_buffer_size`、表缓存、其他杂项等等）可能消耗0.4G，现在剩下13G。考虑到InnoDB Buffer Pool有5%-7%的开销，一个合理的设置是`innodb_buffer_pool_size=12G`，这是我们常在16GB内存的系统中良好运行的设置。

既然我们已经配置了MySQL内存使用，我们还应该看看操作系统配置。第一个问题是，如果您不希望MySQL使用swap，是否应该启用swap？在大多数情况下，答案是肯定的——出于以下两个原因，您希望启用swap（争取最少4GB，不少于安装内存的25%）：
- 当操作系统作为数据库服务器运行时，它很可能有一些未使用的部分。最好是让它swap出这些数据，而不是强迫它将数据保存在内存中。
- 如果您的MySQL配置文件中有误，或者有一些不良进程占据了比预期更多的内存，因使用swap而降低性能要比OOM导致杀死MySQL进程要好的多，OOM可能导致服务中断。

由于我们只希望在紧急情况下使用swap，例如没有可用内存或者swap到空闲进程时，我们希望**降低操作系统使用swap的可能性**（`echo 1 >  /proc/sys/vm/swappiness`）。如果您没有这个配置设置，您可能会发现操作系统swap out了MySQL的部分，只是因为它觉得需要增加可用的文件缓存数量（对MySQL来说，这几乎是一个错误的选择）。

操作系统的下一个设置是OOM。您可能在您的内核日志文件中看到这样的信息：
```
Apr 24 02:43:18 db01 kernel: Out of memory: Kill process 22211 (mysqld) score 986 or sacrifice child
```

当MySQL本身有问题时，这是非常合理的做法。然而，真正的问题可能是您正在运行一些批处理任务：脚本、备份等。在这种情况下，如果系统没有足够的内存（而不是MySQL），您可能希望终止这些进程。

为了使MySQL不太可能因OOM而被杀，您可以调整MySQL的OOM参数，如下所示：
```
echo '-800' > /proc/$(pidof mysqld)/oom_score_adj
```

这将使Linux内核更倾向于杀死其他消耗大量内存的用户。

最后，在一个有多个CPU插槽的系统中，在分配MySQL内存时应该**关心NUMA**。在较新的MySQL版本中，您应该启用`innodb_numa_interleave=1`。在旧版本中，您可以在启动MySQL服务之前手动运行`numactl --interleave=all`，或者在Percona版MySQL中使用`numa_interleave`配置选项。
