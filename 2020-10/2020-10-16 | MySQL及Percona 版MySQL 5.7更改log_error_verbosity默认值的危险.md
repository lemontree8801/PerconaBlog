- [原文链接](https://www.percona.com/blog/2020/10/16/danger-of-changing-default-of-log_error_verbosity-on-mysql-percona-server-for-mysql-5-7/)

# MySQL/Percona 版MySQL 5.7更改`log_error_verbosity`默认值的危险

MySQL/Percona 版MySQL5.7更改`log_error_verbosity`的默认值3有一个隐藏的意想不到的效果！`log_error_verbosity`到底是做什么的？如文档所示：

>系统参数`log_error_verbosity`指明了错误日志中记录事件处理的详细程度。

1仅记录[Errors]；2添加了[Warnings]；3添加了[Notes]部分。

例如，有人可能因为错误日志中记录了太多的[Notes]而更改`log_error_verbosity`的默认值，如：
```
2020-10-08T17:02:56.736179Z 3 [Note] Access denied for user 'root'@'localhost' (using password: NO)
2020-10-08T17:04:48.149038Z 4 [Note] Aborted connection 4 to db: 'unconnected' user: 'root' host: 'localhost' (Got timeout reading communication packets)
```
（P.S. 您可以通过Percona 其他博客了解更多Notes部分）：
[Fixing MySQL 1045 Error: Access Denied](https://www.percona.com/blog/2019/07/05/fixing-a-mysql-1045-error/)
[MySQL “Got an error reading communication packet”](https://www.percona.com/blog/2016/05/16/mysql-got-an-error-reading-communication-packet-errors/)

问题是，`log_error_verbosity`更改为1或者2，MySQL服务启动或者关闭将在日志中没有输出！在出现问题或系统故障时，这确实会使故障排除变得非常困难。5.7启动时，如果`log_error_verbosity`为默认值时，错误日志的完整输出应该为：
```
2020-10-08T16:36:07.302168Z 0 [Note] InnoDB: PUNCH HOLE support available
2020-10-08T16:36:07.302188Z 0 [Note] InnoDB: Mutexes and rw_locks use GCC atomic builtins
...
2020-10-08T16:36:07.303998Z 0 [Note] InnoDB: Initializing buffer pool, total size = 128M, instances = 1, chunk size = 128M
2020-10-08T16:36:07.307823Z 0 [Note] InnoDB: Completed initialization of buffer pool
...
2020-10-08T16:36:07.497571Z 0 [Note] /usr/sbin/mysqld: ready for connections.
```

关闭时：
```
2020-10-08T16:36:10.447002Z 0 [Note] Giving 0 client threads a chance to die gracefully
2020-10-08T16:36:10.447022Z 0 [Note] Shutting down slave threads
2020-10-08T16:36:10.447027Z 0 [Note] Forcefully disconnecting 0 remaining clients
…
2020-10-08T16:36:12.104099Z 0 [Note] /usr/sbin/mysqld: Shutdown complete
```

当`log_error_verbosity`为2时，没有MySQL启动的日志，启动时仅有一些警告输出，如：
```
2020-10-08T16:30:21.966844Z 0 [Warning] TIMESTAMP with implicit DEFAULT value is deprecated. Please use --explicit_defaults_for_timestamp server option (see documentation for more details).
2020-10-08T16:30:22.181367Z 0 [Warning] CA certificate ca.pem is self signed.
2020-10-08T16:30:22.221732Z 0 [Warning] Changed limits: table_open_cache: 431 (requested 6000)
```
如果没有服务重启的相关信息，可以检查系统日志，有重启的相关信息：
```
# cat /var/log/messages 
...
Oct  8 16:31:25 carlos-tutte-latest57-standalone-1 systemd: Starting MySQL Server...
Oct  8 16:31:26 carlos-tutte-latest57-standalone-1 systemd: Started MySQL Server.
```

如果仍然不知道MySQL最近一次什么时候启动，检查状态参数的"Uptime"可以帮助计算最后一次启动。

MySQL/Percona版MySQL8.0没有这个问题，即使`log_error_verbosity`设置为1，如下是启动/关闭时错误日志的输出：
```
2020-10-08T16:31:54.532504Z 0 [System] [MY-010116] [Server] /usr/sbin/mysqld (mysqld 8.0.20-11) starting as process 1052
2020-10-08T16:31:54.540342Z 1 [System] [MY-013576] [InnoDB] InnoDB initialization has started.
2020-10-08T16:31:55.026815Z 1 [System] [MY-013577] [InnoDB] InnoDB initialization has ended.
2020-10-08T16:31:55.136125Z 0 [System] [MY-011323] [Server] X Plugin ready for connections. Socket: '/var/lib/mysql/mysqlx.sock' bind-address: '::' port: 33060
2020-10-08T16:31:55.270669Z 0 [System] [MY-010931] [Server] /usr/sbin/mysqld: ready for connections. Version: '8.0.20-11'  socket: '/var/lib/mysql/mysql.sock'  port: 3306  Percona Server (GPL), Release 11, Revision 5b5a5d2.
2020-10-08T16:32:01.708932Z 0 [System] [MY-010910] [Server] /usr/sbin/mysqld: Shutdown complete (mysqld 8.0.20-11)  Percona Server (GPL), Release 11, Revision 5b5a5d2.
```
总而言之，如果可以的话，避免更改MySQL/Percona版MySQL5.7的`log_error_verbosity`的默认值。如果您需要更改，在线通过`SET GLOBAL`更改，而不是修改配置文件。
