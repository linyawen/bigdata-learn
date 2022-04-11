

# HDFS - HA高可用解决方案-简介

接上一篇文章 "大数据学习之(二)hadoop完全分布式部署"，虽然说已经搭建了完全分布式集群，但是NameNode 却是单点的，一旦故障整个集群都不能访问，而且单节点内存压力也大，现在进行高可用HA 改造。

官方架构图如下：

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



进度 01：17：51