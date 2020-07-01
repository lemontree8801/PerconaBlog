- [原文链接](https://www.percona.com/blog/2020/06/15/evaluating-the-python-tpcc-mongodb-benchmark/)


# 用Python实现MongoDB TPCC 基准压测
我在寻找一个除了可以像YCSB提供的简单键值操作之外，还可以在复杂负载下评估MongoDB性能的工具。

这是为什么论文[Adapting TPC-C Benchmark to Measure Performance of Multi-Document Transactions in MongoDB](http://www.vldb.org/pvldb/vol12/p2254-kamsky.pdf)引起我的注意，并且我尝试了论文中提到的工具[https://github.com/mongodb-labs/py-tpcc](https://github.com/mongodb-labs/py-tpcc)。

顾名思义，Py-tpcc试图模拟TPC-C规范中指定的查询和表结构，并使表结构和查询都适用于MongoDB的事务语言。

我对使用python脚本的基准测试的担心是，Python以高CPU消耗而闻名，我想评估py-tpcc的负载是否能充分利用服务器资源？瓶颈是否不在客户端，而是在服务端？

让我们看一下py-tpcc。

## 硬件规格
对于客户端和服务端，我将使用相同的裸服务器，通过10Gb网络连接。

节点的规格为：
```
# Percona Toolkit System Summary Report ######################
    Hostname | beast-node4-ubuntu
      System | Supermicro; SYS-F619P2-RTN; v0123456789 (Other)
    Platform | Linux
     Release | Ubuntu 18.04.4 LTS (bionic)
      Kernel | 5.3.0-42-generic
Architecture | CPU = 64-bit, OS = 64-bit
# Processor ##################################################
  Processors | physical = 2, cores = 40, virtual = 80, hyperthreading = yes
      Models | 80xIntel(R) Xeon(R) Gold 6230 CPU @ 2.10GHz
      Caches | 80x28160 KB
# Memory #####################################################
       Total | 187.6G
  Swappiness | 0
```

## MongoDB拓扑
对于MongoDB，我使用了一个单节点实例，配置为具有一个节点的非分片的副本集。

**但是首先，顺便说一下，使用PyPy可以让某些部分更好。**

第一步是加载数据到数据库，并且我也想比较各个MongoDB的版本，所以我使用了以下版本：
- Percona版本MongoDB 4.0.8-11
- Percona版本MongoDB 4.2.7-7
- MongoDB社区版 4.4-rc8（测试时4.4最新可用版本）

用py-tpcc加载数据，我们执行以下命令：
```
tpcc.py --config mconfig --warehouses 100 --no-execute mongodb
```
这将加载100个仓库的数据（应该是大约10Gb的原始数据）。第一个惊喜来自该命令的执行时间。

对Percona MongoDB 4.0.8-11版本：
```
time ./tpcc.py --config mconfig --warehouses 100 --no-execute mongodb
2020-06-10 08:55:37,273 [<module>:245] INFO : Initializing TPC-C benchmark using MongodbDriver
2020-06-10 08:55:37,274 [<module>:255] INFO : Loading TPC-C benchmark data using MongodbDriver
an
real    87m45.815s
user    83m3.677s
sys     4m12.884s
```

花费了87分45秒，这实在太长了。

当我们尝试分析它时，我们可以看到服务端实际上是空闲的，而客户端有一个CPU已经100%，这表明瓶颈在客户端生成数据。

**如何改善Python?**

我follow Pypy项目很长一段时间了，所以现在是时候看看使用PyPy能否改善py-tpcc：
```
time /mnt/data/vadim/bench/pypy2.7-v7.3.1-linux64/bin/pypy tpcc.py --config mconfig --warehouses 100 --no-execute mongodb
2020-06-10 12:00:16,647 [<module>:245] INFO : Initializing TPC-C benchmark using MongodbDriver
2020-06-10 12:00:16,647 [<module>:255] INFO : Loading TPC-C benchmark data using MongodbDriver

real    9m34.223s
user    5m29.182s
sys     0m36.051s
```

这是一个很大的提升。现在执行时间是9分34秒，而不是87分45秒。9倍的提升！最终数据库大小为15GB。

在PyPy下所有MongoDB版本加载数据的时间对比：
|版本|加载时间|
|:-:|:-:|
|Percona MongoDB 4.0.18-11|9分34秒|
|Percona MongoDB 4.2.7-7|12分56秒|
|MongoDB 社区版 4.4-rc8|10分21秒|

4.2版本中加载时间变长了，但是在4.4版本中得到了改善。完成数据加载，就可以运行基准压测了。

我将压测900秒，报告显示在此期间执行的**NEW_ORDER**事务的数量。我还测试了并发从10到250的场景。

首先，让我们看一下PyPy能否有所提升。

我用Percona版MongoDB 4.0.18-11进行了测试。结果是每分钟执行**NEW_ORDER**事务的数量。
|并发数|使用Pypy|未使用PyPy|
|:-:|:-:|:-:|
|10|1035|803|
|30|2023|1609|
|50|2651|2111|
|70|3105|2506|
|90|3434|2838|
|110|3656|3092|
|130|3747|3300|
|150|3733|3478|
|170|3682|3600|
|190|3621|3668|
|210|3552|3678|
|230|3502|3635|
|250|3466|3555|


![enter image description here](https://www.percona.com/blog/wp-content/uploads/2020/06/Screen-Shot-2020-06-15-at-12.10.03-PM.png)

PyPy改善了结果，但在本场景中，它并没有提升到加载数据时的9倍。我们也可以比较MongoDB服务端在使用PyPy和没有使用PyPy运行py-tpcc的CPU使用情况。

![enter image description here](https://www.percona.com/blog/wp-content/uploads/2020/06/Screen-Shot-2020-06-15-at-12.10.31-PM.png)

使用PyPy的客户端可以提高服务器的CPU利用率，对高并发的场景下，用户和系统CPU超过了80%。

## Py-tpcc基准压测结果
我继续在PyPy下运行py-tpcc来比较不同MongoDB版本结果：每分钟**NEWORDER**的事务：
|并发|4.0.18|4.2.7|4.4-rc|
|:-:|:-:|:-:|:-:|
|10|1035|1001|1054|
|30|2023|1987|2075|
|50|2651|2663|2678|
|70|3105|3178|3160|
|90|3434|3580|3507|
|110|3656|3928|3765|
|130|3747|4212|3929|
|150|3733|4348|3766|
|170|3682|4171|3824|
|190|3621|4242|3847|
|210|3552|4363|3822|
|230|3502|4028|3726|
|250|3466|4114|3591|

![](https://www.percona.com/blog/wp-content/uploads/2020/06/Screen-Shot-2020-06-15-at-12.11.50-PM.png)

**一些有趣的观察结果：**
- 4.2.7在少量并发下表现较差，在50个并发之后的高并发场景表现较好。
- 结果仅供参考，所有的服务按以下方式启动：
```
numactl --interleave=all ./mongod --dbpath=/data/mongo/ --bind_ip_all --replSet node2 --slowms=10000
```

## 总结
我认为py-tpcc工具为评估MongoDB提供了一个有趣的工作负载，我计划在将来使用它。在PyPy环境下似乎是有提高的，特别是加载数据时。
