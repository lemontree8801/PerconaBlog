# innodb_buffer_pool_size设为内存的80%是正确的吗？
- 现在，任何知道如何调优InnoDB的人，都会将`innodb_buffer_pool_size`设置为物理内存的80%。这是一个非常常见的调优建议，它似乎在许多DBA脑海中根深蒂固。至今的MySQL手册都提到了这条规则，所以谁又能责怪DBA呢？问题是：这样做有意义吗？
## 服务器是如何使用内存的？
- 在我们对这个建议提出疑问之前，让我们回想一下典型的MySQL服务器是如何占用内存的。这个列表不一定完整，但我认为它涵盖了MySQL服务器可能消耗内存的大部分方向。
	- 操作系统：内核，正在运行的进程，文件系统缓存等。
	- MySQL固定使用：query cache，InnoDB buffer pool size，mysqld rss等。
	- MySQL基于负载使用：连接，每个查询的缓冲（join buffer，sort buffer）等。
	- MySQL复制使用：binary log cache，复制连接，Galera gcache和cert index等。
	- 同一服务器上的其他服务：Web服务，缓存服务，定时任务等。
- 对优化InnoDB，`innodb_buffer_pool_size`无疑是最重要的参数。预计它将占用MySQL/Innodb专用服务器上的大部分内存，但当然，其他本地服务可能会影响它的调优方式。如果它（以及服务器上的其他内存消耗）太大，可能会使用Swap并迅速降低性能。
- 此外，MySQL服务器本身的负载可能会导致很多变化。服务器是否有许多打开的链接和活跃查询在消耗内存？由此造成的内存消耗在不同服务器之间可能存在显著差异。
- 最后，像Galera这样的复制机制有自己的内存使用模式，可能需要对缓冲池进行一些调整。
- 我们可以清楚地看到，80%规则并不像现实那样。
## 经验法则
- 但是，为了便于讨论，我们假设80%规则是一个起点。一个帮助我们快速获得调优数值进而让服务器运行。假设我们还不知道关于系统上的负载，但是我们知道系统是专用于InnoDB的，那么我们的80%规则将如何发挥作用？

|服务器总内存|Buffer pool的80%法则|剩余内存|
|-|-|-|
|1G|800MB|200MB|
|16G|13G|3G|
|32G|26G|6G|
|64G|51G|13G|
|128G|102G|26G|
|256G|205G|51G|
|512G|409G|103G|
|1024G|819G|205G|

- 在较低的数值上，80%法则看起来很合理。然而，随着我们的大型服务器，它开始显得不是那么理智。规则的适用，它意味着负载内存消耗与所需buffer pool 大小成正比，但通常并非如此。我们服务器有1TB内存，可能不需要其中205G的内存来处理链接和查询之类的事情（MySQL可能无法处理那么多的活跃链接和查询）
- 所以，如果您真的把所有的钱都花在一个强劲的服务器上，您真的愿意因为这个法则为这个资源支付20%的税吗？
## 法则来源
- 在我第一次MySQL会议中，大概是2006-2007年我在雅虎工作的时候，我参加了由Heikki Tuuri(InnoDB的原作者)和Peter Zaitsev主持的InnoDB调优演讲。我清楚得记得我问过80%法则，因为雅虎有一些强大的64G服务器，而这个法则并不适合我。
- Heikki的回答让我印象深刻。他说（不是直接引用）：“嗯，我测试的服务器有1GB的内存，80%似乎是正确的”。他然后澄清了一点，如果没有内存问题，并表示不适用于更大的服务器。
## 您应如何调优`innodb_buffer_pool_size`?
- 80%可能是一个很好的起点和经验法则。您确实要确保服务器有足够的可用内存用于操作系统和未知的负载。然而，正如之前所述，服务器越大，根据法则越有可能浪费内存。我认为对于大多数人来说，多少都是凭经验决定的，主要是因为在当前版本中更改InnoDB buffer pool需要重启。（5.6版本）
- 那么，更好的经验法则是什么呢？我的原则是，在生产环境中，在不使用swap的情况下，尽量调大`innodb_buffer_pool_size`。原则上这听起来不错，但同样，它需要重新启动，而且说起来容易做起来难。
- 幸运的是，MySQL5.7和它的innodb buffer pool在线更改大小功能可以让这一原则更容易遵循。看到大量的空闲内存（或者使用文件系统缓存率）时，动态调大buffer pool。看到有swap，只需要调低它并不需要重启。在实践中，我怀疑使用该功能时可能会出现一些与性能相关的问题，但这至少是朝着正确方向迈出的一大步。
## 更多内容
### 博客
- [MySQL server memory usage troubleshooting tips](https://www.percona.com/blog/2014/01/24/mysql-server-memory-usage-2/)([中文翻译](https://github.com/lemontree8801/PerconaBlog/blob/master/2014-01/2014-01-24%20%7C%20MySQL%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%86%85%E5%AD%98%E4%BD%BF%E7%94%A8%E6%95%85%E9%9A%9C%E6%8E%92%E9%99%A4%E6%8A%80%E5%B7%A7.md))
- [MySQL 5.7 performance tuning immediately after installation](https://www.percona.com/blog/2016/10/12/mysql-5-7-performance-tuning-immediately-after-installation/)([中文翻译](https://github.com/lemontree8801/PerconaBlog/blob/master/2016-10/2016-10-12%20%7C%20%E5%AE%89%E8%A3%85%E5%90%8EMySQL5.7%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96.md))
- [Ten MySQL performance tuning settings after installation](https://www.percona.com/blog/2014/01/28/10-mysql-performance-tuning-settings-after-installation/)([中文翻译](https://github.com/lemontree8801/PerconaBlog/blob/master/2014-01/2014-01-28%20%7C%20MySQL%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96%E8%AE%BE%E7%BD%AE.md))
### 免费电子书
- Practical MySQL Performance Tuning: section 1 – understanding query parameter tuning and MySQL optimization
- Practical MySQL Performance Tuning: section 2 – troubleshooting performance issues and optimizing MySQL
- Practical MySQL Performance Tuning: section 3 – query optimization
