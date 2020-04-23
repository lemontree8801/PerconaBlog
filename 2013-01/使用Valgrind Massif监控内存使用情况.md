- [原文链接](https://www.percona.com/blog/2013/01/09/profiling-mysql-memory-usage-with-valgrind-massif/)

# 使用Valgrind Massif监控MySQL内存使用情况
- 有时候，您需要**确切**地知道`mysqld`服务（或任何其他程序）使用了多少内存，分配在哪里（为哪个函数），它是如何这么多的（一个回溯），分配发生的时间点。
- 例如：您可能已经注意到，在执行特定的查询之后，内存急剧增加。或者`mysqld`总体上占用了太多内存。或者您可能注意到随着时间推移，`mysqld`占用内存缓慢增加，这表明可能存在内存bug。
- 无论是什么原因，有一个简单但强大的方法来分析MySQL内存使用情况——Valgrind的Massif工具。Massif手册的摘录（堆内存只是分配给程序使用的内存池）；
>Massif不仅告诉您程序正在使用多少堆内存，还提供非常详细的信息，指出程序的哪些部分负责分配堆内存。

- 首先，我们需要得到Valgrind程序。你可以使用操作系统自带的最新版本（yum或者apt-get install Valgrind），我更喜欢获取和编译最新版本（目前是3.8.1）：
```
sudo yum remove valgrind*      # or apt-get etc.
sudo yum install wget make gcc gcc-c++ libtool libaio-devel bzip2 glibc*
wget http://valgrind.org/downloads/valgrind-3.8.1.tar.bz2      # Or newer
tar -xf valgrind-3.8.1.tar.bz2
cd valgrind-3.8.1
./configure
make
sudo make install
valgrind --version             # check version to be same as what was downloaded (3.8.1 here)
```
- 自己编译有以下优点：
	- 1.当使用最新版本的Valgrind时，即使是“开箱即用”的编译（即不做任何更改），您也会发现与早期版本相比，问题更少。例如，早期的版本可能因硬编码具有较小的Valgrind内部存储器跟踪。换句话说，在大缓冲池的情况下，运行Valgrind可能会遇到一些问题。
	- 2.如果您自行编译，如果这些内部阈值限制仍然太小，那么您可以在编译之前轻松修改它们。经常增大的设定是在`coregrind/m_aspacemgr/aspacemgr-linux.c`中的`VG_N_SEGMENTS`（当您看到“Valgrind: FATAL: VG_N_SEGMENTS is too lo”时）
	- 3.新版本更好的支持较新的硬件和软件。
- 一旦“valgrind-version”返回正确安装的版本，就可以开始使用了。在本例中，我们将输出打印到`/tmp/massif.out`中。如果您想打印到另一个目录文件下（因此必须设置正确的文件权限等），请使用：
```
$ touch /your_location/massif.out
$ chown user:group /your_location/massif.out    # Use the user mysqld will now run under (check 'user' setting in my.cnf also) (see <a title="Changing MySQL User" href="http://dev.mysql.com/doc/refman/5.5/en/changing-mysql-user.html" target="_blank" rel="noopener noreferrer">here</a> if this is not clear)
```
- 现在，在Valgrind下运行mysqld之前，请确保有debug symbols。当二进制文件没有删除debug symbols时（下载“GA”[一般可用]包时可能是优化过或者精简之后的二进制文件，它们是为了速度优化而不是为了调试）。如果您的二进制文件已被精简，那么可以使用以下几种方法获取`mysqld`的debug版以用于内存分析：
	- 下载合适的debuginfo包（这些包可能不适用于所有版本）。
	- 下载与当前使用的服务版本相同的debug版二进制文件，只需要简单的用debug版`mysqld`代替当前`mysqld`。（即`shutdown`，` mv mysqld mysqld.old`，`cp /debug_bin_path/mysqld`， `./mysqld, startup`）
	- 如果您有（通过下载或者在以前的存储中）与当前使用的版本相同的可用的源代码，那么只需要调试编译源码，使用`mysqld`二进制文件替换精简版。（如，Percona5.5版源码可以使用`./build/build-binary –debug ..`调试编译）。
- Valgrind Massif需要debug symbol信息存在，以便能够打印堆栈跟踪信息，以显示消耗内存的位置。如果没有可用的debug symbols，您将无法看到负责内存使用的实际函数调用。如果您不确定是否已经精简了二进制文件，那么只需测试以下的过程，看看您得到了什么输出。
- 设置好debug symbols后，标准关闭`mysqld`服务，然后使用Massif工具在Valgrind下手动重启它：
```
$ valgrind --tool=massif --massif-out-file=/tmp/massif.out /path/to/mysqld {mysqld options...}
```
- 注意， `{mysqld options}`可以包括如` –default-file=/etc/my.cnf`（如果这是您的`my.cnf`文件所在的位置），以便将`mysqld`指向您的配置文件等。在`mysqld`正确启动之后（检查是否可以使用`mysql`客户端登陆），您可以执行您认为增加内存使用或者触发内存问题所需的任何步骤。您也可以让服务运行一段时间（例如，如果您经历了内存随时间的增加而增加）。
- 完成之后，关闭`mysqld`（再次使用常规关闭过程），然后使用`ms_print`将`masif.out`输出内存使用情况的文本图：
```
ms_print /tmp/massif.out
```
- 我们最近处理的一个客户问题的部分示例输出：
```
96.51% (<strong>68,180,839B</strong>) (<strong>heap allocation functions</strong>) malloc/new/new[], --alloc-fns, etc.
 -><strong>50.57%</strong> (<strong>35,728,995B</strong>) 0x7A3CB0: <strong>my_malloc</strong> (in /usr/local/percona/mysql-5.5.28/usr/sbin/mysqld)
 | -><strong>10.10%</strong> (<strong>7,135,744B</strong>) 0x7255BB: <strong>Log_event::read_log_event</strong>(char const*, unsigned int, char const**, Format_description_log_event const*) (in /usr/local/percona/mysql-5.5.28/usr/sbin/mysqld)
 | | ->10.10% (7,135,744B) 0x728DAA: Log_event::read_log_event(st_io_cache*, st_mysql_mutex*, Format_description_log_event const*) (in /usr/local/percona/mysql-5.5.28/usr/sbin/mysqld)
 | |   ->10.10% (7,135,744B) 0x5300A8: ??? (in /usr/local/percona/mysql-5.5.28/usr/sbin/mysqld)
 | |   | ->10.10% (7,135,744B) 0x5316EC: handle_slave_sql (in /usr/local/percona/mysql-5.5.28/usr/sbin/mysqld)
 | |   |   ->10.10% (7,135,744B) 0x3ECF60677B: start_thread (in /lib64/libpthread-2.5.so)
 | |   |     ->10.10% (7,135,744B) 0x3ECEAD325B: clone (in /lib64/libc-2.5.so)
 [...]
```
- 之后的一些快照：
```
92.81% (<strong>381,901,760B</strong>) (<strong>heap allocation functions</strong>) malloc/new/new[], --alloc-fns, etc.
-><strong>84.91%</strong> (<strong>349,404,796B</strong>) 0x7A3CB0: <strong>my_malloc</strong> (in /usr/local/percona/mysql-5.5.28/usr/sbin/mysqld)
| -><strong>27.73%</strong> (<strong>114,084,096B</strong>) 0x7255BB: <strong>Log_event::read_log_event</strong>(char const*, unsigned int, char const**, Format_description_log_event const*) (in /usr/local/percona/mysql-5.5.28/usr/sbin/mysqld)
| | ->27.73% (114,084,096B) 0x728DAA: Log_event::read_log_event(st_io_cache*, st_mysql_mutex*, Format_description_log_event const*) (in /usr/local/percona/mysql-5.5.28/usr/sbin/mysqld)
| | ->27.73% (114,084,096B) 0x5300A8: ??? (in /usr/local/percona/mysql-5.5.28/usr/sbin/mysqld)
| | | ->27.73% (114,084,096B) 0x5316EC: handle_slave_sql (in /usr/local/percona/mysql-5.5.28/usr/sbin/mysqld)
| | | ->27.73% (114,084,096B) 0x3ECF60677B: start_thread (in /lib64/libpthread-2.5.so)
| | | ->27.73% (114,084,096B) 0x3ECEAD325B: clone (in /lib64/libc-2.5.so)
```
- 正如您所看到的，`Log_event::read_log_event`函数分配了大量内存（本例中太多）。您还可以从快照中看到分配给函数的内存显著增加。这个案例帮助锁定了一个过滤复制的从节点上内存泄漏的bug（在实际的[bug报告](https://bugs.launchpad.net/percona-server/+bug/1042946)中可以阅读更多内容）。
- 除了按以上方式运行Valgrind Massif外，您还可以更改Massif的快照选项和其他命令行选项，以使快照频率等符合您的特定需求。但是，您可能会发现默认选项在大多数情况下都能很好的运行。
- 更高级的做法是您可以使用Valgrind的`gdbserver`按需获取大量的快照（例如，您可以在执行任何可能显著改变内存使用的命令之前、期间和之后初始化大量的快照）。
