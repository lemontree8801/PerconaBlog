# Percona XtraBackup: 备份和还原单个表或数据库

- 原文连接：[percona-xtrabackup-backup-and-restore-of-a-single-table-or-database](https://www.percona.com/blog/2020/04/10/percona-xtrabackup-backup-and-restore-of-a-single-table-or-database/)

- 备份和恢复一个完整的数据库是一个常见工作。但是，如果您只需要还原由错误的SQL而被错误修改的一个表，这时该怎么办？Percona XtaBackup可以帮助您。
- 为了测试，我们将使用一个由sysbench创建的测试数据库。测试使用8.0版本的Percona XtaBackup和Percona版MySQL。
## 还原单个表
- 这里我们将备份表sbtest2，并恢复它。表的初始checksum值为：
```
8.0.19>CHECKSUM TABLE sbtest2;
+--------------+-----------+
| Table        | Checksum |
+--------------+-----------+
| test.sbtest2 | 905286813 |
+--------------+-----------+
1 row in set (0.01 sec)
```
- 使用`--tables`选项备份单个InnoDB表：
```
./xtrabackup --user=backup --password='Bspass!4%' --backup --tables=sbtest2 --target-dir=$HOME/dbbackup_PS8_table -S $HOME/PS130320_8_0_19_10_debug/socket.sock --datadir=$HOME/PS130320_8_0_19_10_debug/data
```
- XtaBackup将表文件**sbtest2.ibd**与备份所需其他文件一起放到备份目录中（dbbackup_PS8_table/test）。
- 您可以在`--tables`选项中使用匹配规则，XtrBackup将备份所有与规则匹配的表。如果有许多表需要备份，那么可以将这些表写在文本文件中，并使用`--tables-file`选项。还可以使用选项`--tables-exclude`过滤表。
- 现在使用额外的`--export`选项准备备份。这是一个准备表配置的特殊选项。
```
./xtrabackup --prepare --export --target-dir=$HOME/dbbackup_PS8_table
```
- 准备之后，文件`sbtest2.ibd`和`sbtest.cfg`在备份目录中可用。要还原该表，我们必须首先从数据库中删除现有表空间。
```
8.0.19>ALTER TABLE sbtest2 DISCARD TABLESPACE;
Query OK, 0 rows affected (0.20 sec)
```
- 现在，将表文件从备份目录（dbbackup_PS8_table/test/sbtest2.*）复制到Percona Server数据目录（PS130320_8_0_19_10_debug/data/test）。
**注意**：复制文件之前，请禁用selinux。复制文件后，如果备份用户不同，则将复制文件的owner改为mysql用户。
- 最后，导入表空间。
```
8.0.19>ALTER TABLE sbtest2 IMPORT TABLESPACE;
Query OK, 0 rows affected (1.12 sec)
```
- 还原后的表checksum值为：
```
8.0.19>CHECKSUM TABLE sbtest2;
+--------------+-----------+
| Table        | Checksum |
+--------------+-----------+
| test.sbtest2 | 905286813 |
+--------------+-----------+
1 row in set (0.02 sec)
```
- 表成功还原。
- 另一种方法是备份整个数据库，使用备份还原一个或者多张表。这里，备份仅使用`--backup`选项。
```
./xtrabackup --user=backup --password='Bspass!4%' --backup --target-dir=$HOME/dbbackup_PS8 -S $HOME/PS130320_8_0_19_10_debug/socket.sock --datadir=$HOME/PS130320_8_0_19_10_debug/data
```
- 使用`--export`选项准备备份。
```
./xtrabackup --prepare --export --target_dir=$HOME/dbbackup_PS8
```
- 接下来，discard表的表空间，将表文件从备份目录复制到Precona Server数据目录，然后导入表空间。
- 对于MyISAM表，备份和准备过程与上述相同，唯一区别是要删除表，使用`IMPORT TABLE`语句还原该表。
## 还原整个Schema/数据库
- 我们可以备份数据库中的一个schema，使用上述相同的步骤还原它。
- 使用`--databases`选项备份数据库。
```
./xtrabackup --user=backup --password='Bspass!4%' --backup --databases=test --target-dir=$HOME/dbbackup_PS8_db -S $HOME/PS130320_8_0_19_10_debug/socket.sock --datadir=$HOME/PS130320_8_0_19_10_debug/data
```
- 对不止一个数据库，指定数据库列表，如`--databases="db1 db2 db3"`。数据库同样也可以写入一个文本文件，使用`--database-file`选项。过滤备份不用的数据库，使用选项`--databases-exclude`。
- 使用`--export`选项准备备份。
```
./xtrabackup --prepare --export --target-dir=$HOME/dbbackup_PS8_db
```
- 使用`ALTER TABLE <table name> DISCARD TABLESPACE`移除数据库中所有InnoDB表的表空间。
- 拷贝备份目录（dbbackup_PS8_db/test/*）所有文件到MySQL数据目录下（PS130320_8_0_19_10_debug/data/test）。
- **注意**：复制文件之前，请禁用selinux。复制文件后，如果备份用户不同，则将复制文件的owner改为mysql用户。
- 最后，使用`ALTER TABLE <table name> IMPORT TABLESPACE`还原表。
- 这只会将表还原到备份时的状态。对于基于时间点的恢复，可以进一步将binlog应用到数据库，但是应该注意只应用影响正在恢复表的事务。（作为从节点，并行复制&复制过滤，加速恢复）
- 使用这个方法的优点是不需要停数据库服务器。缺点是每个表都要单独恢复，但是可以使用脚本解决这个问题。
## 总结
- 只需要使用几个命令，就可以使用Percona XtaBackup备份和恢复表和数据库。

## 附录：5.7中单表恢复参考
- 为《深入浅出MySQL 第三版》第28章读书笔记

- 应用全备时候，加上`--export`选项，用于生成import table所需的.exp文件
```
innobackupex --defaults-file=/etc/my.cnf --user=root --password= --use=memory=256m --redo-only --apply-log /service/backup/  --export
```
- 重建dept_emp表
```
mysql > create table  dept_emp(xxxxxxxx);
```
- discard 此表
```
mysql > altet table dept_emp discard tablespace;
```
- 复制dept_emp.exp和dept_emp.ibd文件到数据目录
```
cd /service/backup/employees
cp dept_emp.exp dept_emp.ibd /service/databases/mysql/data/
```
- 加载此表
```
mysql > alter table dept_emp import tablespace;
```
## 疑问
-  MySQL8是不需要做apply-log了吗？（需要测试&看一下XtraBackup官档）
