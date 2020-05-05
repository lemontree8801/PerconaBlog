- [原文链接](https://www.percona.com/blog/2020/05/05/setting-up-an-innodb-cluster-with-a-few-lines-of-code/)


# 用几行代码设置InnoDB Cluster
在当今时代，大型企业公司使用Ansible、Puppet或者Chef配置MySQL的副本集或者集群。这减轻了部署和工作流管理的负担。

但是，对一些较小的公司，学习曲线阻碍了立即采用自动化软件。这就是MySQL Shell的用途，它允许您用不到60行的代码部署一个n节点的InnoDB Cluster或者副本集。

## 要求
- 适用于8.0.17版之后的Percona MySQL，最好是8.0.19版本，每个节点以独立服务器启动。
- Percona MySQL Shell 8.0或等效的上游版本
- MySQL `root`用户和密码或者等同的具有赋权的用户
- 每个节点的主机名已被配置在`/etc/hosts`中。
```
10.84.148.18 clusternode3
10.84.148.199 clusternode2
10.84.148.214 clusternode1
```

执行脚本并提供需要的数据：
```
[root@clientServer mysqlsh]# mysqlsh -f setup_innodb_cluster.js
InnoDB cluster set up
==================================
Setting up a Percona Server for MySQL - InnoDB cluster.

Password for the MySQL root account: **********
Number of data nodes: 3
Hostname for node1: clusternode1
Hostname for node2: clusternode2
Hostname for node3: clusternode3
```
之后，脚本会完成配置3节点InnoDB cluster剩下的过程，您只需要等待它完成。

我们可以看到每个节点用`dba.configureInstance()`函数配置。这个阶段我们同样创建了clusterAdmin用户和密码。
```
Configuring the instances.
=> Configuring MySQL instance at clusternode1:3306 for use in an InnoDB cluster...

This instance reports its own address as clusternode1:3306
Clients and other cluster members will communicate with it through this address by default. If this is not correct, the report_host MySQL system variable should be changed.

NOTE: Some configuration options need to be fixed:
+-----------------+---------------+----------------+----------------------------+
| Variable        | Current Value | Required Value | Note                       |
+-----------------+---------------+----------------+----------------------------+
| binlog_checksum | CRC32         | NONE           | Update the server variable |
+-----------------+---------------+----------------+----------------------------+

Cluster admin user 'pscluster'@'cluster%' created.
Configuring instance...
The instance 'clusternode1:3306' was configured to be used in an InnoDB cluster.

=> Configuring MySQL instance at clusternode2:3306 for use in an InnoDB cluster...

This instance reports its own address as clusternode2:3306
Clients and other cluster members will communicate with it through this address by default. If this is not correct, the report_host MySQL system variable should be changed.

NOTE: Some configuration options need to be fixed:
+-----------------+---------------+----------------+----------------------------+
| Variable        | Current Value | Required Value | Note                       |
+-----------------+---------------+----------------+----------------------------+
| binlog_checksum | CRC32         | NONE           | Update the server variable |
+-----------------+---------------+----------------+----------------------------+

Cluster admin user 'pscluster'@'cluster%' created.
Configuring instance...
The instance 'clusternode2:3306' was configured to be used in an InnoDB cluster.

=> Configuring MySQL instance at clusternode3:3306 for use in an InnoDB cluster...

This instance reports its own address as clusternode3:3306
Clients and other cluster members will communicate with it through this address by default. If this is not correct, the report_host MySQL system variable should be changed.

NOTE: Some configuration options need to be fixed:
+-----------------+---------------+----------------+----------------------------+
| Variable        | Current Value | Required Value | Note                       |
+-----------------+---------------+----------------+----------------------------+
| binlog_checksum | CRC32         | NONE           | Update the server variable |
+-----------------+---------------+----------------+----------------------------+

Cluster admin user 'pscluster'@'cluster%' created.
Configuring instance...
The instance 'clusternode3:3306' was configured to be used in an InnoDB cluster.

Configuring Instances completed.
```

在这一阶段，InnoDB Cluster从第一个节点开始初始化。此时脚本执行`dba.createCluster()`函数。
```
Setting up InnoDB Cluster.

A new InnoDB cluster will be created on instance 'clusternode1:3306'.

Validating instance configuration at clusternode1:3306...

This instance reports its own address as clusternode1:3306

Instance configuration is suitable.
NOTE: Group Replication will communicate with other members using 'clusternode1:33061'. Use the localAddress option to override.

Creating InnoDB cluster 'InnoDBCluster' on 'clusternode1:3306'...

Adding Seed Instance...
Cluster successfully created. Use Cluster.addInstance() to add MySQL instances.
At least 3 instances are needed for the cluster to be able to withstand up to
one server failure.
```

接下来是向集群添加另2个实例。如您所示，它使用CLONE插件从第一个节点拷贝数据。这里我们使用`<Cluster>.addInstance()`函数向集群添加每个实例。
```
Adding instance to the cluster...

Monitoring recovery process of the new cluster member. Press ^C to stop monitoring and let it continue in background.
Clone based state recovery is now in progress.

NOTE: A server restart is expected to happen as part of the clone process. If the
server does not support the RESTART command or does not come back after a
while, you may need to manually start it back.

* Waiting for clone to finish...
NOTE: clusternode2:3306 is being cloned from clusternode1:3306
** Stage DROP DATA: Completed



** Clone Transfer      
FILE COPY  ############################################################  100%  Completed    
PAGE COPY  ############################################################  100%  Completed    
REDO COPY  ############################################################  100%  Completed

** Stage RECOVERY: \
NOTE: clusternode2:3306 is shutting down...

* Waiting for server restart... ready
* clusternode2:3306 has restarted, waiting for clone to finish...
* Clone process has finished: 59.61 MB transferred in about 1 second (~1.00 B/s)

Incremental state recovery is now in progress.

* Waiting for distributed recovery to finish...
NOTE: 'clusternode2:3306' is being recovered from 'clusternode1:3306'
* Distributed recovery has finished

The instance 'clusternode2:3306' was successfully added to the cluster.


=>
WARNING: A GTID set check of the MySQL instance at 'clusternode3:3306' determined that it contains transactions that do not originate from the cluster, which must be discarded before it can join the cluster.

clusternode3:3306 has the following errant GTIDs that do not exist in the cluster:
2442b908-7356-11ea-b46b-00163e1f9fe5:1-2

WARNING: Discarding these extra GTID events can either be done manually or by completely overwriting the state of clusternode3:3306 with a physical snapshot from an existing cluster member. To use this method by default, set the 'recoveryMethod' option to 'clone'.

Having extra GTID events is not expected, and it is recommended to investigate this further and ensure that the data can be removed prior to choosing the clone recovery method.
Clone based recovery selected through the recoveryMethod option

NOTE: Group Replication will communicate with other members using 'clusternode3:33061'. Use the localAddress option to override.

Validating instance configuration at clusternode3:3306...

This instance reports its own address as clusternode3:3306

Instance configuration is suitable.
A new instance will be added to the InnoDB cluster. Depending on the amount of
data on the cluster this might take from a few seconds to several hours.

Adding instance to the cluster...

Monitoring recovery process of the new cluster member. Press ^C to stop monitoring and let it continue in background.
Clone based state recovery is now in progress.

NOTE: A server restart is expected to happen as part of the clone process. If the
server does not support the RESTART command or does not come back after a
while, you may need to manually start it back.

* Waiting for clone to finish...
NOTE: clusternode3:3306 is being cloned from clusternode2:3306
** Stage DROP DATA: Completed



** Clone Transfer      
FILE COPY  ############################################################  100%  Completed    
PAGE COPY  ############################################################  100%  Completed    
REDO COPY  ############################################################  100%  Completed

** Stage RECOVERY: \
NOTE: clusternode3:3306 is shutting down...

* Waiting for server restart... ready
* clusternode3:3306 has restarted, waiting for clone to finish...
* Clone process has finished: 59.62 MB transferred in about 1 second (~1.00 B/s)

State recovery already finished for 'clusternode3:3306'

The instance 'clusternode3:3306' was successfully added to the cluster.


Instances successfully added to the cluster.

InnoDB cluster deployed successfully.
```
现在，我们已经有了一个三节点的InnoDB Cluster，只需不到五分钟，不到六十行代码。

这是源码：
```
print('InnoDB cluster set up\n');
print('==================================\n');
print('Setting up a Percona Server for MySQL - InnoDB cluster.\n\n');

var dbPass = shell.prompt('Password for the MySQL root account: ', { type: "password" });
var numNodes = shell.prompt('Number of data nodes: ');
var dbHosts = [];

for (let i = 1; i <= numNodes; i++) {
    var hostName = shell.prompt('Hostname for node' + i + ': ');
    dbHosts.push(hostName);
}

function sleep(milliseconds) {
    const date = Date.now();
    let currentDate = null;
    do {
        currentDate = Date.now();
    } while (currentDate - date < milliseconds);
}

print('\nNumber of Hosts: ' + dbHosts.length + '\n');
print('\nList of hosts:\n');
for (let s = 0; s < dbHosts.length; s++) {
    print('Host: ' + dbHosts[s] + '\n');
}

function setupCluster() {
    print('\nConfiguring the instances.');
    for (let n = 0; n < dbHosts.length; n++) { print('\n=> ');
        dba.configureInstance('root@' + dbHosts[n] + ':3306', { clusterAdmin: "", clusterAdminPassword: '', password: dbPass, interactive: false, restart: true });
    }
    print('\nConfiguring Instances completed.\n\n');

    sleep(5000); // source: https://www.sitepoint.com/delay-sleep-pause-wait/

    print('Setting up InnoDB Cluster.\n\n');
    shell.connect({ user: 'root', password: dbPass, host: dbHosts[0], port: 3306 });

    var cluster = dba.createCluster("InnoDBCluster");

    print('Adding instances to the cluster.\n');
    for (let x = 1; x < dbHosts.length; x++) { print('\n=> ');
        cluster.addInstance('root@' + dbHosts[x] + ':3306', { password: dbPass, recoveryMethod: 'clone' });
    }
    print('\nInstances successfully added to the cluster.\n');
}

try {
    setupCluster();

    print('\nInnoDB cluster deployed successfully.\n');
} catch (e) {
    print('\nThe InnoDB cluster could not be created.\n');
    print(e + '\n');
}
```
您也可以使用Python创建类似的脚本，因为MySQL Shell也支持它。
