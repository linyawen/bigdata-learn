

# HDFS - HA高可用-简介

接上一篇文章 "大数据学习之(二)hadoop完全分布式部署"，虽然说已经搭建了完全分布式集群，但是NameNode 却是单点的，一旦故障整个集群都不能访问，而且单节点内存压力也大，现在进行高可用HA 改造。

## 官方架构图

![hdfs-ha_0](https://s2.loli.net/2022/04/10/b3sB4kOPUdx5CXG.png)

其实原理很简单，3点:

1. 再加一台NameNode2，跟原来的NameNode1通过JournalNodes集群同步数据(ap,最终一致性)。
2. 每个NameNode机器上各自启动一个辅助进程FailoveController，这两个FailoveController连到zookeeper 进行分布式抢锁，比如FC1 抢到了锁，就把本机的NameNode 设置为主，把另一台机器的NameNode设置为从（通过ssh进行设置，所以两台NameNode要配置免密）。
3. FailoveController 的职责其实就是 实现 主备2个NameNode的故障自动切换，只有 active的提供服务。

PS 疑问：

- journalNodes 看起来是不是跟 zookeeper很像？但是不一样，zk侧重分布式协调（不适合存太多数据），journalNode侧重分布式存储，用来同步NameNode的editlog 比较合适。
- 为什么hadoop的作者不把FailoveController的功能直接集成到NameNode里？简单的功能拆出多个进程不是增加运行时故障的场景吗，还有部署复杂度也增加了。难道 是为了避免nameNode的兼容性问题？

# 一、角色规划

标准为1的为本次HA要增加部署的，主要是 JournalNode(JNN),FailoverController(FC),Zookeeper(ZK)

| host  | NN   | NN   | JNN  | SNN  | DN   | FC   | ZK   |
| ----- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| node1 | *    |      | 1    |      |      | 1    |      |
| node2 |      | 1    | 1    | *    | *    | 1    | 1    |
| node3 |      |      | 1    |      | *    |      | 1    |
| node4 |      |      |      |      | *    |      | 1    |



# 二、环境 配置

## 1.关调集群

sh stop-dfs.sh

```
sh stop-dfs.sh
```

## 2.两个NameNode之间ssh免密 

之前在单机伪分布式部署和完全分布式部署中，node1已经把公钥分发给node1~node4。所以现在也要把node2的公钥分发给node1~node4。

(1).登录到node2

```
# 生成密钥并分发给自己
ssh-keygen -t dsa -P '' -f ~/.ssh/id_dsa
cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys
#分发给其他node1
scp ./id_dsa.pub  node1:`pwd`/node2.pub
```

(2).登录到node1

```
cat node2.pub >> authorized_keys
```

## 3.zk集群搭建

 在node2，node3，node4 搭建zk集群

### 3.1登录node2

```
cd /opt/bigdata/zoo....
cd conf
cp zoo_sample.cfg  zoo.cfg
#修改数据目录，声明集群，设置权重
vi zoo.cfg
	datadir=/var/bigdata/hadoop/zk
	server.1=node2:2888:3888
	server.2=node3:2888:3888
	server.3=node4:2888:3888
mkdir /var/bigdata/hadoop/zk
echo 1 >  /var/bigdata/hadoop/zk/myid 
#加到环境变量
vi /etc/profile
	export ZOOKEEPER_HOME=/opt/bigdata/zookeeper-3.4.6
	export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$ZOOKEEPER_HOME/bin
source /etc/profile
```

```
#分发zk包到 node3，node4
cd /opt/bigdata
scp -r ./zookeeper-3.4.6  node3:`pwd`
scp -r ./zookeeper-3.4.6  node4:`pwd`
```

### 3.2登录node3修改zk

```
mkdir /var/bigdata/hadoop/zk
echo 2 >  /var/bigdata/hadoop/zk/myid 
#加到环境变量
vi /etc/profile
	export ZOOKEEPER_HOME=/opt/bigdata/zookeeper-3.4.6
	export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$ZOOKEEPER_HOME/bin
source /etc/profile
```

### 3.3登录node4修改zk

```
mkdir /var/bigdata/hadoop/zk
echo 3 >  /var/bigdata/hadoop/zk/myid 
#加到环境变量
vi /etc/profile
	export ZOOKEEPER_HOME=/opt/bigdata/zookeeper-3.4.6
	export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$ZOOKEEPER_HOME/bin
source /etc/profile
```

### 3.4启动zk集群

依次在node2,3,4中执行

```
zkServer.sh start
```

ps: 一般情况下，server.2 (node3)为leader，因为启动到node3时刚好过半，可以选主了。

## 4.修改hadoop配置文件

进行HA 集群改造的响应配置修改，关键词：zk，failoverController，journalNode

### 4.1 vi core-site.xml

```
<configuration>
        <property>
                <name>fs.defaultFS</name>
                <value>hdfs://mycluster</value>
        </property>
        <property>
           <name>ha.zookeeper.quorum</name>
           <value>node2:2181,node3:2181,node4:2181</value>
         </property>
</configuration>
```

###  4.2  vi hdfs-site.xml

ps: 

- 去掉secondNameNode，职责由NameNodeStandBy代替
- 存储路径要修改，之前完全分布式放在full下，现在full改成ha

```
<configuration>
#以下是  一对多，逻辑到物理节点的映射
		<property>
		  <name>dfs.nameservices</name>
		  <value>mycluster</value>
		</property>
		<property>
		  <name>dfs.ha.namenodes.mycluster</name>
		  <value>nn1,nn2</value>
		</property>
		<property>
		  <name>dfs.namenode.rpc-address.mycluster.nn1</name>
		  <value>node1:8020</value>
		</property>
		<property>
		  <name>dfs.namenode.rpc-address.mycluster.nn2</name>
		  <value>node2:8020</value>
		</property>
		<property>
		  <name>dfs.namenode.http-address.mycluster.nn1</name>
		  <value>node1:50070</value>
		</property>
		<property>
		  <name>dfs.namenode.http-address.mycluster.nn2</name>
		  <value>node2:50070</value>
		</property>

		#以下是JN在哪里启动，数据存那个磁盘
		<property>
		  <name>dfs.namenode.shared.edits.dir</name>
		  <value>qjournal://node1:8485;node2:8485;node3:8485/mycluster</value>
		</property>
		# 修改存储路径full为ha
		<property>
		  <name>dfs.journalnode.edits.dir</name>
		  <value>/var/bigdata/hadoop/ha/dfs/jn</value>
		</property>
		
		#HA角色切换的代理类和实现方法，我们用的ssh免密
		<property>
		  <name>dfs.client.failover.proxy.provider.mycluster</name>
		  <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
		</property>
		<property>
		  <name>dfs.ha.fencing.methods</name>
		  <value>sshfence</value>
		</property>
		<property>
		  <name>dfs.ha.fencing.ssh.private-key-files</name>
		  <value>/root/.ssh/id_dsa</value>
		</property>
		
		#开启自动化： 启动zkfc
		 <property>
		   <name>dfs.ha.automatic-failover.enabled</name>
		   <value>true</value>
		 </property>
</configuration>
```

### 4.3分发配置

```
scp core-site.xml hdfs-site.xml   node2:`pwd`
scp core-site.xml hdfs-site.xml   node3:`pwd`
scp core-site.xml hdfs-site.xml   node4:`pwd`
```

## 5.初始化hdfs

### 5.1 启动journalnode

```
#在node1，node2,node3 依次执行
hadoop-daemon.sh start journalnode
```

### 5.2 格式化dfs

找一台namenode执行如下命令

```
#node1中
hdfs namenode -format
#此时node1中name目录文件初始化完毕，node2,node3,node4中journalNode相关的jn目录文件也初始化完毕。 
```

### 5.3 启动这个格式化的NameNode 

5.2格式化的是node1，所以这一步也要启动node1的NameNode，以备另一台同步。

```
#node1中
hadoop-daemon.sh start namenode
```

### 5.4 配置备用NameNode 进行同步 

备用的NameNode在 node2

```
#node2中
hdfs namenode -bootstrapStandby
#此时node2中/var/bigdata/hadoop/ha/dfs/name目录文件初始化完毕
```

**此时hdfs初始化完成。**

## 6.初始化FailoverController

### 6.1 格式化zk

```
#node1中执行（只有第一次搭建做）。
hdfs zkfc  -formatZK
```

### 6.2 检查zk

```
#node2,3,4 是zk集群，选一台执行
zkCli.sh
#然后查看zk状态
[zk: localhost:2181(CONNECTED) 0] ls /
[zookeeper, hadoop-ha]
[zk: localhost:2181(CONNECTED) 1] ls /hadoop-ha
[mycluster]
[zk: localhost:2181(CONNECTED) 2] ls /hadoop-ha/mycluster
[]
[zk: localhost:2181(CONNECTED) 3] 
#可以看到zk里面已经有了hadoop-ha节点 ，但是mycluster里没有数据（namenode1,namenode2）
```

## 7.启动dfs

```
#node1
start-dfs.sh	
```

执行后，会在所有节点拉起各个组件(在简介-官方架构图中所示)，如下：

```
[root@node1 current]# start-dfs.sh 
Starting namenodes on [node1 node2]
node1: namenode running as process 1640. Stop it first.
node2: starting namenode, logging to /opt/bigdata/hadoop-2.6.5/logs/hadoop-root-namenode-node2.out
node2: starting datanode, logging to /opt/bigdata/hadoop-2.6.5/logs/hadoop-root-datanode-node2.out
node4: starting datanode, logging to /opt/bigdata/hadoop-2.6.5/logs/hadoop-root-datanode-node4.out
node3: starting datanode, logging to /opt/bigdata/hadoop-2.6.5/logs/hadoop-root-datanode-node3.out
Starting journal nodes [node1 node2 node3]
node1: journalnode running as process 1482. Stop it first.
node2: journalnode running as process 1766. Stop it first.
node3: journalnode running as process 1576. Stop it first.
Starting ZK Failover Controllers on NN hosts [node1 node2]
node1: starting zkfc, logging to /opt/bigdata/hadoop-2.6.5/logs/hadoop-root-zkfc-node1.out
node2: starting zkfc, logging to /opt/bigdata/hadoop-2.6.5/logs/hadoop-root-zkfc-node2.out
```

在zk中，也有了主备nameNode的分布式锁节点:

```
#node4
[zk: localhost:2181(CONNECTED) 3] ls /hadoop-ha/mycluster
[ActiveBreadCrumb, ActiveStandbyElectorLock]

#注意，可以看到现在是node1拿到了锁 
[zk: localhost:2181(CONNECTED) 8] get /hadoop-ha/mycluster/ActiveStandbyElectorLock

	myclusternn1node1 �>(�>
cZxid = 0x10000000a
ctime = Wed Apr 13 02:15:38 CST 2022
mZxid = 0x10000000a
mtime = Wed Apr 13 02:15:38 CST 2022
pZxid = 0x10000000a
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x3801ec3f1470002
dataLength = 29
numChildren = 0


```

jourlnalNode中也开始同步数据如:

```
[root@node3 current]# pwd
/var/bigdata/hadoop/ha/dfs/jn/mycluster/current
[root@node3 current]# ll
total 1052
-rw-r--r--. 1 root root       8 Apr 13 02:21 committed-txid
-rw-r--r--. 1 root root      42 Apr 13 02:17 edits_0000000000000000001-0000000000000000002
-rw-r--r--. 1 root root      42 Apr 13 02:19 edits_0000000000000000003-0000000000000000004
-rw-r--r--. 1 root root      42 Apr 13 02:21 edits_0000000000000000005-0000000000000000006
-rw-r--r--. 1 root root 1048576 Apr 13 02:21 edits_inprogress_0000000000000000007
-rw-r--r--. 1 root root       2 Apr 13 02:15 last-promised-epoch
-rw-r--r--. 1 root root       2 Apr 13 02:15 last-writer-epoch
drwxr-xr-x. 2 root root       6 Apr 13 01:50 paxos
-rw-r--r--. 1 root root     154 Apr 13 01:50 VERSION

```

访问node1的dfs 控制台页面可以看到状态为active http://127.0.0.1:50070/dfshealth.html#tab-overview（我这里本机50070端口转发到node1虚拟机的50070端口）

如果访问node1的dfs 控制台页面则可以看到状态为standby。

![hadoop-ha-node1-dashboard](https://s2.loli.net/2022/04/13/koZglVL6PEUsfi4.png)

## 8.重启dfs集群

插曲：如果你测试的虚拟机集群重新开机后，怎么启动整个集群呢：

### 1.启动zk集群

```
#node2,3,4
zkServer.sh start
```

```
[zk: localhost:2181(CONNECTED) 8] ls  /hadoop-ha/mycluster 
[ActiveBreadCrumb]
```

### 2.启动journalnode

```
#在node1，node2,node3 依次执行
hadoop-daemon.sh start journalnode
```

### 3.启动dfs

```
#node1
start-dfs.sh	
```

## 9. 故障模拟

模拟各种故障，验证HA 高可用性

### 9.1 模拟主NameNode 挂掉

可以看到当前node1的nameNode为主

```
#node2,3,4随便一台
zkCli.sh
[zk: localhost:2181(CONNECTED) 11] get /hadoop-ha/mycluster/ActiveStandbyElectorLock

	myclusternn1node1 �>(�>
cZxid = 0x200000007

```

kill -9 掉node1的nameNode进程，再检查一次zk，看到node2的nameNode抢到了主。（此时是zkfc监控到了变化，主动进行切换）

```
#node2,3,4
[zk: localhost:2181(CONNECTED) 12] get /hadoop-ha/mycluster/ActiveStandbyElectorLock

	myclusternn2node2 �>(�>

```

 我这本机50072端口映射到 虚拟机node2的50070端口，所以这么访问node2的hdfs管理后台![hadoop-ha-node2-dashboard](https://s2.loli.net/2022/04/17/YsmWhDPzZ27xd9O.png) 

ps：试验完记得把node1的nameNode重启回来

```
#node1中执行启动命令，此时 node1为从。
hadoop-daemon.sh start namenode
```

### 9.2 模拟zkfc挂掉

类似的kill  抢锁进程 FailoverController (即zkfc)，检查zk，发现主从切换（此时是利用了zk的临时节点的原理）。

恢复zkfc

```
hadoop-daemon.sh start zkfc
```

### 9.3 模拟网络不通

关掉网卡

```
#node2 目前是主，再node2处理
#我的网卡名是 enp0s3，具体可以ifconfig命令查，一有的是 eth0
ifconfig enp0s3 down
```

观察zk，锁已经被node1抢走

```
#node3，4
[zk: localhost:2181(CONNECTED) 1] get /hadoop-ha/mycluster/ActiveStandbyElectorLock

	myclusternn1node1 �>(�>
```

但是!!! node1 居然还没有切成主！！！因为node1上的zkfc尝试着ssh去探测node2，发现网络不通所以没有切，避免脑裂的发生。（此时node1没法通过ssh把node2置为从）

![hadoop-ha-node1-dashboard3](https://s2.loli.net/2022/04/17/Z8goaIQnyJpTqVS.png)

重启网卡,看到node1 顺利切成了主，node2切成从

```
#node2
ifconfig enp0s3 up
```

![hadoop-ha-node1-dashboard4](https://s2.loli.net/2022/04/17/wyWCPhmSkN2J3E9.png)