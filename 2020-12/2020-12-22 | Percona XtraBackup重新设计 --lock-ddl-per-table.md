
[Redesign of -lock-ddl-per-table in Percona XtraBackup - Percona Database Performance Blog](https://www.percona.com/blog/2020/12/22/redesign-of-lock-ddl-per-table-in-percona-xtrabackup/)

MySQL5.7，以及许多其他的改进，对创建索引带来了巨大的负担(详见[WL#7277]([https://dev.mysql.com/worklog/task/?id=7277](https://dev.mysql.com/worklog/task/?id=7277)))

[WL#7277: InnoDB: Bulk Load for Create Index](https://dev.mysql.com/worklog/task/?id=7277)

通过禁用redo记录日志并直接对表空间文件修改来添加索引。这个更改对备份工具需要特别注意。为了阻塞实例上的DDL语句，Percona版MySQL采用`LOCK TABLES FOR BACKUP`。Percona XtraBackup 在备份期间使用加这个锁。这个锁不影响DML语句。

MySQL 5.7没有选项阻塞实例DDL，并允许DML。因此，Percona XtraBackup也实现了`—lock-ddl-per-table` 。在讨论其他细节之前，让我们先了解一下`—lock-ddl-per-table` 到目前为止的工作方式：

1. PXB启动，在checkpoint标记后，分析和复制所有的redo日志。
2. Fork专用线程记录之后的新的redo日志变化。
3. 获得需要复制的所有表空间列表。
4. 遍历这个表空间列表，对每个表空间，它处理如下：
    - 查询 `INFORMATION_SCHEMA.INNODB_SYS_TABLES`，如果是8.0的服务端，查询`INFORMATION_SCHEMA.INNODB_TABLES` 检查哪些表属于这个表空间ID，如果是共享表空间则对上面的表加元数据锁。
    - 复制表空间的.ibd文件。

此方法有一前提条件，如果记录redo线程遇到 `MLOG_INDEX_LOAD` 事件(大批量加载数据操作生成的Redo日志需要通知备份工具已经忽略了redo日志对数据文件的修改)，则由于元数据锁表空间可以安全地被复制，表空间的 `MLOG_INDEX_LOAD` 事件也会被复制。

这个前提并不总是正确，可能导致备份不一致；以下是一些例子：

## 全文索引

全文索引有自己的表空间：

```sql
mysql> SELECT table_id, name, space from INFORMATION_SCHEMA.INNODB_SYS_TABLES WHERE name LIKE '%FTS%';
+----------+----------------------------------------------------+-------+
| table_id | name                                               | space |
+----------+----------------------------------------------------+-------+
|     1169 | test/FTS_000000000000002e_0000000000000508_INDEX_1 |  1157 |
|     1170 | test/FTS_000000000000002e_0000000000000508_INDEX_2 |  1158 |
|     1171 | test/FTS_000000000000002e_0000000000000508_INDEX_3 |  1159 |
|     1172 | test/FTS_000000000000002e_0000000000000508_INDEX_4 |  1160 |
|     1173 | test/FTS_000000000000002e_0000000000000508_INDEX_5 |  1161 |
|     1174 | test/FTS_000000000000002e_0000000000000508_INDEX_6 |  1162 |
|     1175 | test/FTS_000000000000002e_BEING_DELETED            |  1163 |
|     1176 | test/FTS_000000000000002e_BEING_DELETED_CACHE      |  1164 |
|     1177 | test/FTS_000000000000002e_CONFIG                   |  1165 |
|     1178 | test/FTS_000000000000002e_DELETED                  |  1166 |
|     1179 | test/FTS_000000000000002e_DELETED_CACHE            |  1167 |
+----------+----------------------------------------------------+-------+
11 rows in set (0.01 sec)
```

对于当前的方法，Percona XtraBackup将尝试在FTS_000000000000002e_0000000000000508_INDEX_1 上运行一个SELECT，这不是我们能做的事情。此时FTS所属的基础表可能受到元数据锁的保护，也可能没有，这可能导致FTS索引在没有保护的情况下被复制。

全文索引有一个定义好的命名模式，在上述情况下，我们可以通过将`FTS_` 后的前16个字符从十六进制转换为十进制，很容易地提取表ID，这将给我们FTS所属的底层表：

```sql
session 1> CREATE FULLTEXT INDEX full_index on joinit2 (s);
session 2> SELECT table_id, name, space from INFORMATION_SCHEMA.INNODB_SYS_TABLES WHERE table_id >= 1319;
+----------+----------------------------------------------------+-------+
| table_id | name                                               | space |
+----------+----------------------------------------------------+-------+
|     1321 | test/#sql-ib1320-2000853746                        |  1309 |
|     1322 | test/FTS_0000000000000529_00000000000005b6_INDEX_1 |  1310 |
|     1323 | test/FTS_0000000000000529_00000000000005b6_INDEX_2 |  1311 |
|     1324 | test/FTS_0000000000000529_00000000000005b6_INDEX_3 |  1312 |
|     1325 | test/FTS_0000000000000529_00000000000005b6_INDEX_4 |  1313 |
|     1326 | test/FTS_0000000000000529_00000000000005b6_INDEX_5 |  1314 |
|     1327 | test/FTS_0000000000000529_00000000000005b6_INDEX_6 |  1315 |
|     1328 | test/FTS_0000000000000529_BEING_DELETED            |  1316 |
|     1329 | test/FTS_0000000000000529_BEING_DELETED_CACHE      |  1317 |
|     1330 | test/FTS_0000000000000529_CONFIG                   |  1318 |
|     1331 | test/FTS_0000000000000529_DELETED                  |  1319 |
|     1332 | test/FTS_0000000000000529_DELETED_CACHE            |  1320 |
|     1320 | test/joinit2                                       |  1308 |
+----------+----------------------------------------------------+-------+
```

FTS_0000000000000529 转换为table_id 1321，但是，正如您在上面看到的，问题中的FTS属于一个临时表。这是因为在第一次创建FTS时，它必须重新构建表来创建FTS_DOC_ID列。同样，不可能在元数据锁保护下复制文件。

## 在备份过程中添加新表

由于是按表进行元数据锁获取，如果PXB收集表空间列表之后创建了表，那么.ibd文件不会被复制。相反，在`—prepare` 阶段基于添加到redo日志的数据重新构建。如果在根据redo信息重建表之后跳过更改，那么表将是不完整的。

## 共享表空间

`—lock-ddl-per-table` 函数解析共享表空间时，它会得到表空间上创建的所有表的列表，并且这些表都需要元数据锁，然而，没有表空间级别的元数据锁，这意味着无法阻塞在这个表空间上创建一个新表。如果表空间已经被复制，那么它将遵循前面的步骤，并在`—prepare` 阶段通过解析redo日志重新创建。

## 最好/最坏场景

这种不一致备份的结果可能是未知的。最好的情况是，您在备份或者prepare阶段时服务端崩溃。是的，崩溃是最好的情况，因为您会立刻注意到这个问题。

最坏的情况是，数据已丢失，您也没有注意到。这里我不解释了，直接实验。

下载最新版MySQL5.7或者Percona分支版，并下载2.4.21之前的Percona XtraBackup2.4(也可以下载MySQL/PSMySQL8和PXB8)。

为了重现这个场景，我们将使用`gdb` 在PXB备份的特定某个时间暂停执行，或者是备份一张大表，需要一段时间来复制。

```sql
gdb xtrabackup ex 'set args --backup --lock-ddl-per-table --target-dir=/tmp/backup' -ex 'b mdl_lock_table' -ex 'r'
```

这会启动备份。在一个独立的会话，连接MySQL并执行：

```sql
CREATE DATABASE a;
USE a;
CREATE TABLE tb1 (ID INT PRIMARY KEY, name CHAR(1));
INSERT INTO tb1 VALUES (3,'c'), (4, 'd'), (5, 'e');
CREATE INDEX n_index ON tb1(name);
```

在gdb会话，执行

```sql
disa 1
c
quit
```

您的备份将完成。您可以在MySQL上prepare和还原。现在尝试使用以下SQL查询`a.tb1`

```sql
USE a;
SELECT * FROM tb1;
SELECT * FROM tb1 FORCE INDEX(PRIMARY);
SELECT * FROM tb1 FORCE INDEX(n_index);
SELECT * FROM tb1 WHERE name = 'd';
```

以下为结果：

```sql
mysql> SELECT * FROM tb1;
Empty set (0.00 sec)

mysql> SELECT * FROM tb1 FORCE INDEX(PRIMARY);
+----+------+
| ID | name |
+----+------+
|  3 | c    |
|  4 | d    |
|  5 | e    |
+----+------+
3 rows in set (0.00 sec)

mysql> SELECT * FROM tb1 FORCE INDEX(n_index);
Empty set (0.00 sec)

mysql> SELECT * FROM tb1 WHERE name = 'd';
Empty set (0.00 sec)
```

如您所见，使用主键进行查询表明数据确实存在。索引`n_index` 存在，但是对应的值不存在，导致了结果不一致，这可能会在您发现问题之前造成很大危害。

## 重新设计 `—lock-ddl-per-table`

考虑到以上几点，我们重构了`—lock-ddl-per-table` ，以保证一致性的备份。Percona XtraBackUp 2.4.21和Percona XtraBackUp8.0.22主要变化如下：

- 在备份开始时就加MDL锁，在第一步/抓取redo日志之前。
- 然后在第一次扫描redo log时，假使在备份开始前有创建索引，我们仍然可以记录`MLOG_INDEX_LOAD` 事件。现在，解析和接收仍然是安全的。
- 一旦第一次扫描完成，负责跟踪新的redo log事件的线程就会启动，如果遇到`MLOG_INDEX_LOAD` 事件，这个线程将停止备份。
- 收集要复制的表空间列表。
- 启动备份真实的.ibd文件。

还进行了其他的优化：

- 如果.ibd文件属于临时表，将跳过运行`SELECT`的尝试，因为`SELECT`查询将永远无法工作。
- 因为我们在复制单个文件之前使用MDL锁，假如我们要处理FTS，我们也会跳过锁。对于FTS，我们最终将(或已经)在基表上获取一个元数据锁，这样就可以安全地跳过这些文件。
- 使用MDL的查询已经改为不检索任何数据，因为`SELECT 1 FROM table LIMIT 0`就足以获取表上的MDL锁。
- —lock-ddl-per-table是针对WL#7227所述的在备份过程中所做的更改的一种变通方法。在Percona Server 5.7中，我们实现了LOCK TABLES FOR BACKUP(通过PXB的`—lock-ddl`参数执行)，它获得实例级的MDL锁。如果用户在Percona版MySQL5.7中使用`—lock-ddl-per-table` ，会有警告抛出，建议您使用`—lock-ddl` 来代替。
- 对于MySQL8，upstream已经实现了`LOCK INSTANCE FOR BACKUP` ，它的工作原理类似于Percona Server的 LOCK TABLES FOR BACKUP，在Percona XtraBackUp8中，—lock-ddl-per-table已经被废弃，它仍然可以工作，但也会发出警告，建议用户切换到`—lock-ddl` 。

## 总结

正如文中所述，WL#7277带来了一些真正的挑战，它们会以不同的方式影响备份，其中一些不容易发现。Percona XtraBackUp总是支持一致性，并且已经进行了增强，以便在可能的情况下(过程中尽早使用元数据锁)绕过这些限制，并在不可避免地出现不一致时停止备份。

Percona版MySQL用户和MySQL 8应该总是使用—lock-ddl，对于这些类型的情况，这是一个更健壮和安全的锁。
