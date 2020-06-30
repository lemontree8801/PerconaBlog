- [原文链接](https://www.percona.com/blog/2020/06/30/analyzing-mysql-with-strace/)

# 使用strace分析MySQL
在这篇博客文章中，我们将简要地探讨操作系统工具`strace`。由于会影响性能，它并没有被广泛使用，我们不建议您在生产环境中使用它。不过，它在帮助您理解MySQL中发生的一些事情，涉及操作系统，以及作为故障排除的终极大招，还是非常有帮助的。

`strace`工具拦截并记录被追踪进程执行的所有系统调用和接收的系统信号。它非常适用于复杂的故障排除，但是要注意的是，它对被追踪的进程性能影响很大。

我们用一个简单的问题举例：在MySQL中执行`FLUSH LOGS`，在操作系统中打开的文件是什么？我们可以查看文档，但我们决定使用`strace`找到答案。

为此，我们启动了一个MySQL测试实例，并用以下命令：
```
strace -s2048 -f -o /tmp/strace.out -p $(pgrep -x mysqld)
```

`-s`选项是输出打印的字符串的行数。默认值是32，这在大多数情况下是不够的，所以我们使用2000，它可以让我们在日志中看到更多行。

`-f`选项是追踪子进程。这一点很重要，因为MySQL是基于线程的架构，而且我们要分析的连接（运行`FLUSH LOGS`）是主进程的子进程。

`-o`选项用于指定输出文件，`-p`选项指定`strace`追踪的PID（本例中为`mysqld`的PID）。

首先，我们要找到会话对应的系统线程ID。基于此，我们可以过滤`strace.out`文件，我们没必要在输出文件中有过多噪音。

我们可以通过`SHOW PROCESSLIST`和`PERFORMANCE_SCHEMA.THREADS`找到我们的线程ID：
```
mysql> show processlist;
+----+------+-----------------+------+---------+------+----------+------------------+-----------+---------------+
| Id | User | Host | db | Command | Time | State | Info | Rows_sent | Rows_examined |
+----+------+-----------------+------+---------+------+----------+------------------+-----------+---------------+
| 5 | root | localhost:47724 | test | Query | 0 | starting | show processlist | 0 | 0 |
+----+------+-----------------+------+---------+------+----------+------------------+-----------+---------------+
1 row in set (0.00 sec)

mysql> select THREAD_OS_ID from performance_schema.threads where PROCESSLIST_ID = 5;
+--------------+
| THREAD_OS_ID |
+--------------+
| 1851 |
+--------------+
1 row in set (0.00 sec)
```

现在我们得到了系统线程ID，我们可以在同一会话中执行`FLUSH lOGS`:
```
mysql> flush logs;
Query OK, 0 rows affected (0.16 sec)
```
我们用线程ID`grep`一下`strace.out`文件，寻找我们想要查看的打开和关闭的系统调用（用于打开和关闭文件）：
```
shell> grep 1851 /tmp/strace.out | egrep "open|close"

1851 open("/home/vinicius.grippa/sandboxes/rsandbox_5_7_22/master/data/msandbox.err", O_WRONLY|O_CREAT|O_APPEND, 0666) = 32
1851 close(32) = 0
1851 open("/home/vinicius.grippa/sandboxes/rsandbox_5_7_22/master/data/msandbox.err", O_WRONLY|O_CREAT|O_APPEND, 0666) = 32
1851 close(32) = 0
1851 openat(AT_FDCWD, "./", O_RDONLY|O_NONBLOCK|O_DIRECTORY|O_CLOEXEC) = 32
1851 close(32) = 0
1851 close(41) = 0
1851 close(4) = 0
1851 open("./mysql-bin.index", O_RDWR|O_CREAT, 0640) = 4
1851 open("./mysql-bin.~rec~", O_RDWR|O_CREAT, 0640) = 32
1851 close(32) = 0
1851 open("./mysql-bin.~rec~", O_RDWR|O_CREAT, 0640) = 32
1851 open("./mysql-bin.000013", O_WRONLY|O_CREAT, 0640) = 41
1851 open("./mysql-bin.index_crash_safe", O_RDWR|O_CREAT, 0640) = 42
1851 close(42) = 0
1851 close(4) = 0
1851 open("./mysql-bin.index", O_RDWR|O_CREAT, 0640) = 4
1851 close(32) = 0
```

在第一行中，我们看到了数字1851，产生系统调用的线程ID（我们用于`grep`），系统调用（打开）的文件`/home/vinicius.grippa/sandboxes/rsandbox_5_7_22/master/data/msandbox.err`，实验实例配置的错误日志路径。我们还发现打开了MySQL二进制日志索引文件（`mysql-bin.index`），`mysql-bin.~rec~`文件，当前MySQL二进制日志文件（`mysql-bin.000013`）,`mysql-bin.index_crash_safe`文件，最后再次打开`mysql-bin.index`。

测试中不出意外的展示了错误日志，当前二进制日志文件，二进制日志索引文件。`mysql-bin.~rec~`文件和`mysql-bin.index_crash_safe`文件一开始让人惊讶，但当您稍微研究一下，就会明白它们的用途。

快速浏览一下“创建”这两个文件的源码：
```
int MYSQL_BIN_LOG::set_purge_index_file_name(const char* base_file_name)
{
    int error = 0;
    DBUG_TRACE;
    if (fn_format(
            purge_index_file_name, base_file_name, mysql_data_home, ".~rec~",
            MYF(MY_UNPACK_FILENAME | MY_SAFE_PATH | MY_REPLACE_EXT))
        == nullptr) {
        error = 1;
        LogErr(ERROR_LEVEL, ER_BINLOG_FAILED_TO_SET_PURGE_INDEX_FILE_NAME);
    }
    return error;
}
```

```
int MYSQL_BIN_LOG::set_crash_safe_index_file_name(const char* base_file_name)
{
    int error = 0;
    DBUG_TRACE;
    if (fn_format(crash_safe_index_file_name, base_file_name, mysql_data_home,
            ".index_crash_safe", MYF(MY_UNPACK_FILENAME | MY_SAFE_PATH | MY_REPLACE_EXT))
        == nullptr) {
        error = 1;
        LogErr(ERROR_LEVEL, ER_BINLOG_CANT_SET_TMP_INDEX_NAME);
    }
    return error;
}
```

函数`fn_format`用于格式化文件名。它们都是辅助文件，可以帮助安全地修改二进制日志索引文件。

## 总结
总之，`strace`是一个强大的工具，它为您提供关于MySQL行为（关于系统调用）的未经过滤的信息。因为它输出了大量的信息，所以可能让您毫无头绪，但如果您知道您要找的是什么，它就会非常有用。最后要注意的是，它会严重降低性能，应该避免在生产环境中使用。
