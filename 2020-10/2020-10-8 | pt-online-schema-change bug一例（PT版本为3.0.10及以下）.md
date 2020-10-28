# pt-online-schema-change bug一例（PT版本为3.0.10及以下）

最近，我在处理一个由我们的支持客户提出的意外问题，在做表结构变更之后，数据变得不一致。

经过调查，发现受影响的表在一些列的注释上有一个特殊的单词，这触发了一个由Percona Toolkit的`TableParser.pm`库引起的已知的（并且已经修复的）的问题。客户使用的是过时的PT工具，`pt-online-schema-change`使用的是有bug的解析器。

**这个bug只会在Percona Toolkit3.0.10以下的版本**，所以如果您已经安装了3.0.11或者更新的版本，您可以跳过博文的剩下部分，因为您并不会受影响。

我写这篇博文是警告`pt-online-schema-change`并且未升级的的使用者，因为这个问题可能是非常危险的，可能会导致数据丢失。

这个问题会表现在两方面。第一个虽然令人困惑，但并不真正危险，因为操作被取消了。当带有错误注释的列不允许空值时，就会发生这种情况。例如：

```
CREATE TABLE `test_not_null` (
`id` int NOT NULL,
`add_id` int NOT NULL COMMENT 'my generated test case',
PRIMARY KEY (`id`)
) ENGINE=InnoDB;
```
表结构变更操作如下所示：
```
$ ./pt-online-schema-change-3.0.10 u=msandbox,p=msandbox,h=localhost,S=/tmp/mysql_sandbox5735.sock,D=test,t=test_not_null --print --alter "engine=InnoDB" --execute
(...)
Altering `test`.`test_not_null`...
Creating new table...
CREATE TABLE `test`.`_test_not_null_new` (
`id` int(11) NOT NULL,
`add_id` int(11) NOT NULL COMMENT 'my generated test case',
PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1
Created new table test._test_not_null_new OK.
Altering new table...
ALTER TABLE `test`.`_test_not_null_new` engine=InnoDB
Altered `test`.`_test_not_null_new` OK.
2020-09-30T21:25:22 Creating triggers...
2020-09-30T21:25:22 Created triggers OK.
2020-09-30T21:25:22 Copying approximately 3 rows...
INSERT LOW_PRIORITY IGNORE INTO `test`.`_test_not_null_new` (`id`) SELECT `id` FROM `test`.`test_not_null` LOCK IN SHARE MODE /*pt-online-schema-change 1438 copy table*/
2020-09-30T21:25:22 Dropping triggers...
DROP TRIGGER IF EXISTS `test`.`pt_osc_test_test_not_null_del`
DROP TRIGGER IF EXISTS `test`.`pt_osc_test_test_not_null_upd`
DROP TRIGGER IF EXISTS `test`.`pt_osc_test_test_not_null_ins`
2020-09-30T21:25:22 Dropped triggers OK.
2020-09-30T21:25:22 Dropping new table...
DROP TABLE IF EXISTS `test`.`_test_not_null_new`;
2020-09-30T21:25:22 Dropped new table OK.
`test`.`test_not_null` was not altered.
2020-09-30T21:25:22 Error copying rows from `test`.`test_not_null` to `test`.`_test_not_null_new`: 2020-09-30T21:25:22 <strong>Copying rows caused a MySQL error 1364:</strong>
Level: Warning
Code: 1364
Message: Field 'add_id' doesn't have a default value
Query: INSERT LOW_PRIORITY IGNORE INTO `test`.`_test_not_null_new` (`id`) SELECT `id` FROM `test`.`test_not_null` LOCK IN SHARE MODE /*pt-online-schema-change 1438 copy table*/
```

因此，操作失败的原因可能不清楚，但至少没有数据损坏。当带注释的列允许空值时，会出现更糟的结果：
```
CREATE TABLE `test_null` (
`id` int NOT NULL,
`add_id` int DEFAULT NULL COMMENT 'my generated test case',
PRIMARY KEY (`id`)
) ENGINE=InnoDB;

mysql [localhost:5735] {msandbox} (test) > select * from test_null;
+----+--------+
| id | add_id |
+----+--------+
| 1  |      1 |
| 2  |      2 |
| 3  |      3 |
+----+--------+
3 rows in set (0.01 sec)
```
对于这个，表结构变更没有报错：
```
$ ./pt-online-schema-change-3.0.10 u=msandbox,p=msandbox,h=localhost,S=/tmp/mysql_sandbox5735.sock,D=test,t=test_null --print --alter "engine=InnoDB" --execute
(...)
Altering `test`.`test_null`...
Creating new table...
CREATE TABLE `test`.`_test_null_new` (
`id` int(11) NOT NULL,
`add_id` int(11) DEFAULT NULL COMMENT 'my generated test case',
PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1
Created new table test._test_null_new OK.
Altering new table...
ALTER TABLE `test`.`_test_null_new` engine=InnoDB
Altered `test`.`_test_null_new` OK.
2020-09-30T21:28:11 Creating triggers...
2020-09-30T21:28:11 Created triggers OK.
2020-09-30T21:28:11 Copying approximately 3 rows...
INSERT LOW_PRIORITY IGNORE INTO `test`.`_test_null_new` (`id`) SELECT `id` FROM `test`.`test_null` LOCK IN SHARE MODE /*pt-online-schema-change 3568 copy table*/
2020-09-30T21:28:11 Copied rows OK.
2020-09-30T21:28:11 Analyzing new table...
2020-09-30T21:28:11 Swapping tables...
RENAME TABLE `test`.`test_null` TO `test`.`_test_null_old`, `test`.`_test_null_new` TO `test`.`test_null`
2020-09-30T21:28:11 Swapped original and new tables OK.
2020-09-30T21:28:11 Dropping old table...
DROP TABLE IF EXISTS `test`.`_test_null_old`
2020-09-30T21:28:11 Dropped old table `test`.`_test_null_old` OK.
2020-09-30T21:28:11 Dropping triggers...
DROP TRIGGER IF EXISTS `test`.`pt_osc_test_test_null_del`
DROP TRIGGER IF EXISTS `test`.`pt_osc_test_test_null_upd`
DROP TRIGGER IF EXISTS `test`.`pt_osc_test_test_null_ins`
2020-09-30T21:28:11 Dropped triggers OK.
Successfully altered `test`.`test_null`.
```
但是，表数据已经不一致：
```
mysql [localhost:5735] {msandbox} (test) > select * from test_null;
+----+--------+
| id | add_id |
+----+--------+
|  1 |   NULL |
|  2 |   NULL |
|  3 |   NULL |
+----+--------+
3 rows in set (0.00 sec)
```
总之，确保您使用的是最新的PT工具包，特别是`pt-online-schema-change`，避免潜在的灾难。在我写这篇文章的时候，最新的稳定版本是3.2.1，针对这个bug的修正版本3.0.11是在2018年7月发布的。
