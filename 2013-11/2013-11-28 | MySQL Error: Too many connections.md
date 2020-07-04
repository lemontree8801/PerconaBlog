- [原文链接](https://www.percona.com/blog/2013/11/28/mysql-error-too-many-connections/)


# MySQL Error：Too many connections
在Percona支持时，我们总是收到关于如何避免可怕的"Too many connections"错误，以及`max_connections`的建议值是多少的问题。因此，在本文中，我将尝试分享这些问题的最佳答案，以便其他人能不再遇到类似的问题。

我的同事Aurimas曾经写过一篇很棒的文章[changing max_connections value via GDB](https://www.percona.com/blog/2010/03/23/too-many-connections-no-problem/)，当MySQL服务端遇到"Too many connections"报错时，无需重启MySQL。

默认情况下，151是MySQL5.5中允许的最大并发客户端连接数。如果您达到`max_connections`的限制，在您试图连接MySQL服务端时，您会遇到"Too many connections"的报错。这意味着所有可用连接都在被其他客户端使用。

MySQL在`max_connections`限制之上允许一个额外的连接，这个连接是为有`SUPER`权限的数据库用户保留的，用于诊断连接问题。通常管理员用户具有`SUPER`权限。您应该避免为应用账户赋予`SUPER`权限。

MySQL中每个客户端连接使用一个线程，大量的活跃线程是性能杀手。通常，大量执行并发查询会导致查询速度显著下降，并增大了死锁的概率。在MySQL5.5之前，它的伸缩性不是很好，虽然MySQL从那时起变得越来越好——但是，如果您有数百个活跃连接在实际执行查询（这不包括休眠[空闲]连接），那么内存使用将会增加。每个连接都需要线程缓冲区。此外，隐式内存表需要更多的内存以及全局缓冲区的内存需求。最重要的是，`tmp_table_size/max_heap_table_size`是每个连接可以使用的，尽管它们不会立刻分配给每个连接。

## 连接数过多
大多数情况下，过多的连接是由于应用程序错误导致没有正确地关闭连接，或者设计错误造成，比如建立了MySQL连接，但应用程序在关闭MySQL处理连接之前忙于做其他的事情。在应用程序没有正确关闭连接时，`wait_timeout`是调优和关闭未使用或者闲置的连接的重要参数，从而降低到MySQL服务端的活跃连接数——这将最终帮助避免"Too many connections"的报错。尽管有些系统即使有大量线程连接也能正常运行，但大多数连接都是空闲的。通常，休眠线程不会占用太多内存——512KB或者更少。`Thread_running`是一个很有价值的监控指标，因为它不计算休眠线程的数量，它显示活跃和当前正在处理的查询数量，而`threads_connected`状态参数显示所有已连接的线程值，包括空闲连接。Peter写了一篇很好的博文，您可以参阅[Is your MySQL Server Loaded ?](https://www.percona.com/blog/2010/03/19/is-your-mysql-server-loaded/)

如果在应用程序端使用连接池，那么数据库端`max_connections`应该大于应用端`max_connections`。如果需要大量连接，连接池也是一个不错的选择。

## max_connections的建议值
这取决于可用内存的大小和每个连接使用多少内存。增加`max_connections`值增加了mysqld服务所需的文件描述符的数量。注意：设置最大的`max_connections`没有硬性限制。因此，您必须根据您的负载、同时连接到MySQL服务端的连接数等明智地选择`max_connections`。一般来说，不建议设置过高的`max_connections`值，因为在某些情况下，如果所有这些连接运行会出现巨大的争用问题，那么可能会导致锁等待或者速度下降。

活跃连接使用临时表或者内存表的情况下，内存使用可能会更高。在内存较小的系统或者在应用程序端有硬连接限制，我们可以使用较小的`max_connections`值，比如100～300。内存为16G时，可以将`max_connections`设置为1000，每个连接缓冲区应该有良好的默认值。而在某些系统中，我们可以看到高达8000的最大连接，但这样的系统通常会在负载峰值时宕机。

为了解决这个问题，Oracle和MariaDB团队实现了线程池。Percona版MySQL从MariaDB移植了这个功能。使用恰当配置的线程池，对于至少数千个并发连接，对某些类型的负载，吞吐量并不会降低。

注意：在MySQL5.6中，如果将`max_connections`值设置过大将占用大量内存。参阅bug报告[http://bugs.mysql.com/bug.php?id=68514](http://bugs.mysql.com/bug.php?id=68514)

## 总结
没有固定的规则来设置恰当的`max_connections`，因为它取决于您的负载。要考虑到每个连接的线程需要内存和昂贵的上下文切换。我建议根据负载为`max_connections`设置一个合理的数值，并尽量避免同时打开太多连接，以便应用程序正常运行。
