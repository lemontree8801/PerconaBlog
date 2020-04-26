- [原文链接](https://www.percona.com/blog/2020/04/24/warning-innodb-difficult-to-find-free-blocks-in-the-buffer-pool/)


# [警告] InnoDB ：很难在Buffer Pool中找到空闲块
- 几周前，我们的一个客户联系我们，询问他们的MySQL错误日志中的警告信息。一段时间后，其他一些客户也发出一些请求，询问是否需要担心这些消息。在这篇文章中，我将介绍将此警告消息写入日志的条件，并解释幕后的一些基本原理。
- 查看MySQL错误日志中出现警告的消息。它说很难在缓冲池中找到一个空闲块，并且在缓冲池中循环搜索了336次。这看起来很奇怪，为什么要循环这么多次？让我们尝试理解这一点。
>[Warning] InnoDB: Difficult to find free blocks in the buffer pool (336 search iterations)! 0 failed attempts to flush a page! Consider increasing the buffer pool size. It is also possible that in your Unix version fsync is very slow, or completely frozen inside the OS kernel. Then upgrading to a newer version of your operating system may help. Look at the number of fsyncs in diagnostic info below. Pending flushes (fsync) log: 0; buffer pool: 0. 1621989850 OS file reads, 1914021664 OS file writes, 110701569 OS fsyncs. Starting InnoDB Monitor to print further diagnostics to the standard output.
>
>2019-10-26T15:02:03.962059Z 4520929 [Warning] InnoDB: Difficult to find free blocks in the buffer pool (337 search iterations)! 0 failed attempts to flush a page! Consider increasing the buffer pool size. It is also possible that in your Unix version fsync is very slow, or completely frozen inside the OS kernel. Then upgrading to a newer version of your operating system may help. Look at the number of fsyncs in diagnostic info below. Pending flushes (fsync) log: 0; buffer pool: 0. 1621989850 OS file reads, 1914021664 OS file writes, 110701569 OS fsyncs. Starting InnoDB Monitor to print further diagnostics to the standard output.

- 简而言之，当InnoDB中的一个线程不断寻找空闲块，但无法从缓冲池中获取一个空闲块的时候，就会出现这些消息。实际上，它并不是那么简单，并且之前还发生了一些事情，了解缓冲池的构建块会使您更容易理解这个案例。缓冲池主要有3个列表组成：
	- 1.空闲列表 ===> 线程可以从磁盘中读入页的页块列表。
	- 2.LRU列表 ===> 最近最少使用的数据/索引页列表。
	- 3.刷新列表 ===> 脏页或修改页列表
- 如您所知，每个查询都是由InnoDB中一个线程处理的。如果所需的页面在缓冲池中不可用，则必须从磁盘获取并放在其中，为此需要从空闲列表中分配一个空闲块。请记住，只能从空闲列表中分配空闲块。因此，需要空闲块的线程需要以下迭代：
	- 从空闲列表中查找空闲块，并在空闲块可用时分配它。如果没有，继续。
	- 扫描LRU列表寻找一个干净的块，并将其移动到空闲列表中，然后使用它。如果没有，继续。
	- 从LRU列表的尾部刷新一个脏页，将空闲块移动到空闲列表，并使用。
- 我把上面的步骤简化了一些，使它更容易理解，但这并不是确切的过程，但肯定是类似的。在任何时候，如果块被移动到一个空闲列表，任何其他线程都可以使用它，这可能使当前线程无法获取空闲块。然后它继续执行循环操作，每次休眠10毫秒。

## 何时将消息写入日志？
- 当一个线程超过20次迭代时，它会将此警告消息写入日志。下面是来自最新版本的代码块，它指出了确切的情况。
```
  if (n_iterations > 20 && srv_buf_pool_old_size == srv_buf_pool_size) {    <strong><======= Here it is</strong>
    ib::warn(ER_IB_MSG_134)
        << "Difficult to find free blocks in the buffer pool"
           " ("
        << n_iterations << " search iterations)! " << flush_failures
        << " failed attempts to"
           " flush a page! Consider increasing the buffer pool"
           " size. It is also possible that in your Unix version"
           " fsync is very slow, or completely frozen inside"
           " the OS kernel. Then upgrading to a newer version"
           " of your operating system may help. Look at the"
           " number of fsyncs in diagnostic info below."
           " Pending flushes (fsync) log: "
```

## 您需要了解什么？
- 这个情况可能是由不同的原因造成的，可以验证下面的列表来找到确切的原因。尽管IO子系统多次饱和，但它不可能对每个实例和负载都是一样的。
	- 慢或者饱和的IO子系统，无法按需要将页面刷新到磁盘。
	- 没有足够的脏页清理线程，无法执行足够的检查点活动，从而无法在缓冲池中创建空闲的页面块。
	- 较小的缓冲池，最终导致较小的空闲列表和LRU列表。
- 类似PMM这样的监控工具对检查上述方面非常有用。从我本地PMM GUI看到的屏幕快照中，您可以看到缓冲池中的空闲页面在不同的时间点几乎是空的，这是导致上述情况的原因之一。这些数据可以从**MySQL InnoDB Metrics** Dashboard的**InnoDB Buffer Pages**面板获得。
![enter image description here](https://www.percona.com/blog/wp-content/uploads/2020/04/Blog1.png)

- 类似地，还有其他面板如**InnoDB Buffer Pool Requests**，可以查看磁盘读取，这是判断当前缓冲池是否足够的另一个指标。通过查看**OS/System Overview** Dashboard的**CPU Usage**面板查看是否有IO wait。下面是一些最常见的避免这种情况的行动计划。
	- 1.当缓冲池中空闲缓冲区总是小于总容量的5%时，增加InnoDB缓冲池的大小。
	- 2.调优/升级IO子系统，比当前更快地完成刷新到磁盘的活动。
	- 3.优化查询，避免全表/索引全扫描，以减少磁盘读入缓冲池的次数。
