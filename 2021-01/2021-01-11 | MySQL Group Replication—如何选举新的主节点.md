- [原文链接](https://www.percona.com/blog/2021/01/11/mysql-group-replication-how-to-elect-the-new-primary-node/)

这篇博文中，我将说明在MySQL组复制中选择主节点的不同方法。在MySQL8.0.2之前，主节点选举基于成员的UUID，在故障转移时，较小的UUID节点会被选举为新主节点。

从MySQL8.0.2开始：我们可以使用服务器权重(`group_replication_member_weight`)选择节点提升为主。这个可以在当前主节点故障时候完成。

从MySQL8.0.12开始，我们可以使用函数`group_replication_set_as_primary`提升任意节点为主节点。这个可以在任意节点故障，任意时间设置。

## 场景

3节点集群，版本为Percona MySQL 8.0.22。

```bash
mysql> select member_host,member_state,member_role,member_version from performance_schema.replication_group_members;
+---------------+--------------+-------------+----------------+
| member_host   | member_state | member_role | member_version |
+---------------+--------------+-------------+----------------+
| 172.28.128.15 | ONLINE       | SECONDARY   | 8.0.22         |
| 172.28.128.14 | ONLINE       | PRIMARY     | 8.0.22         |
| 172.28.128.13 | ONLINE       | SECONDARY   | 8.0.22         |
+---------------+--------------+-------------+----------------+
3 rows in set (0.03 sec)
```

我计划对当前主节点 "**172.28.128.14**" 做系统补丁工作。当我对当前主节点做降级的同时，我需要将 "**172.28.128.15**" 提升为主节点。我们看一下如何通过以下两种方法实现。

- 使用服务器权重(`group_replication_member_weight`)
- 使用函数"`group_replication_set_as_primary`"

## 使用服务器权重(`group_replication_member_weight`)

这种方法并不简单。当当前主节点停止工作时，新节点将提升为主节点。目前，所有节点权重相同。

```bash
[root@innodb1 ~]# mysql -e "select @@hostname, @@group_replication_member_weight\G"
*************************** 1. row ***************************
                       @@hostname: 172.28.128.13
@@group_replication_member_weight: 50

[root@innodb2 ~]#  mysql -e "select @@hostname, @@group_replication_member_weight\G"
*************************** 1. row ***************************
                       @@hostname: 172.28.128.14
@@group_replication_member_weight: 50

[root@innodb3 ~]# mysql -e "select @@hostname, @@group_replication_member_weight\G"
*************************** 1. row ***************************
                       @@hostname: 172.28.128.15
@@group_replication_member_weight: 50
```

我将增加 "**172.28.128.15**" 的权重，这样在 "**172.28.128.14**" 降级时，它将被选举为主节点。

在 "**172.28.128.15**" ，

```bash
mysql> set global group_replication_member_weight = 70;
Query OK, 0 rows affected (0.00 sec)

mysql> select @@group_replication_member_weight;
+-----------------------------------+
| @@group_replication_member_weight |
+-----------------------------------+
|                                70 |
+-----------------------------------+
1 row in set (0.00 sec)
```

注意：设置完权重后，您不需要执行`STOP/START GROUP_REPLICATION`。

"**172.28.128.15**" 的权重增加到70。现在我将对当前主节点 "**172.28.128.14**" 降级。

在 "**172.28.128.14**" ，

```bash
mysql> stop group_replication;
Query OK, 0 rows affected (4.29 sec)

mysql> select member_host,member_state,member_role,member_version from performance_schema.replication_group_members;
+---------------+--------------+-------------+----------------+
| member_host   | member_state | member_role | member_version |
+---------------+--------------+-------------+----------------+
| 172.28.128.14 | OFFLINE      |             |                |
+---------------+--------------+-------------+----------------+
1 row in set (0.01 sec)
```

节点离开集群。

在**"172.28.128.15"**，

```bash
mysql> select member_host,member_state,member_role,member_version from performance_schema.replication_group_members;
+---------------+--------------+-------------+----------------+
| member_host   | member_state | member_role | member_version |
+---------------+--------------+-------------+----------------+
| 172.28.128.15 | ONLINE       | PRIMARY     | 8.0.22         |
| 172.28.128.13 | ONLINE       | SECONDARY   | 8.0.22         |
+---------------+--------------+-------------+----------------+
2 rows in set (0.04 sec)
```

您可以看到"172.28.128.15"被选为新的主节点。

## 使用函数"`group_replication_set_as_primary`"

方法很简单，不需要降级当前主节点来切换主节点。

```bash
+---------------+--------------+-------------+----------------+
| member_host   | member_state | member_role | member_version |
+---------------+--------------+-------------+----------------+
| 172.28.128.15 | ONLINE       | SECONDARY   | 8.0.22         |
| 172.28.128.14 | ONLINE       | PRIMARY     | 8.0.22         |
| 172.28.128.13 | ONLINE       | SECONDARY   | 8.0.22         |
+---------------+--------------+-------------+----------------+
3 rows in set (0.00 sec)
```

现在，我们执行函数"group_replication_set_as_primary"并加上成员的UUID。

在"172.28.128.15"，

```bash
mysql> show global variables like 'server_uu%';
+---------------+--------------------------------------+
| Variable_name | Value                                |
+---------------+--------------------------------------+
| server_uuid   | c5aed435-d58d-11ea-bb26-5254004d77d3 |
+---------------+--------------------------------------+
1 row in set (0.00 sec)

mysql> select group_replication_set_as_primary('c5aed435-d58d-11ea-bb26-5254004d77d3');
+--------------------------------------------------------------------------+
| group_replication_set_as_primary('c5aed435-d58d-11ea-bb26-5254004d77d3') |
+--------------------------------------------------------------------------+
| Primary server switched to: c5aed435-d58d-11ea-bb26-5254004d77d3         |
+--------------------------------------------------------------------------+
1 row in set (1.03 sec)

mysql> select member_host,member_state,member_role,member_version from performance_schema.replication_group_members;
+---------------+--------------+-------------+----------------+
| member_host   | member_state | member_role | member_version |
+---------------+--------------+-------------+----------------+
| 172.28.128.15 | ONLINE       | PRIMARY     | 8.0.22         |
| 172.28.128.14 | ONLINE       | SECONDARY   | 8.0.22         |
| 172.28.128.13 | ONLINE       | SECONDARY   | 8.0.22         |
+---------------+--------------+-------------+----------------+
3 rows in set (0.00 sec)
```

"172.28.128.15"被选举为一个主节点。

注意：你可以在集群上的任一节点执行函数"`group_replication_set_as_primary_node`"。

这个方法非常有用，由于某原因或者想切换到高配服务器上，你希望切换主节点，它将非常有用。
