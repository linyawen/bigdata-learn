# 部署配置

完全分布式,四节点

## 一、角色规划

- name node 存文件的元数据，切片信息等吃内存，所以单独部署
- SNN随便
- data node 三台，决定了集群容量

| host  | NN   | SNN  | DN   |
| ----- | ---- | ---- | ---- |
| node1 | *    |      |      |
| node2 |      | *    | *    |
| node3 |      |      | *    |
| node4 |      |      | *    |

基于上一个笔记 "**大数据学习之(一) hadoop伪分布式搭建试用**"的虚拟机环境node1，继续进行搭建。

## 二、环境搭建

## node1:

```
sh stop-dfs.sh
```

## node2~node4基础环境准备

新建另外3台虚拟机(node2~node4) ，参考node1，进行基础设施的设置好 ip，主机名，关闭防火墙和 selinux，时间同步, 安装jdk， JAVA_HOME、HADOOP_HOME等。

| host  | ip            |
| ----- | ------------- |
| node1 | 192.168.130.3 |
| node2 | 192.168.130.4 |
| node3 | 192.168.130.6 |
| node4 | 192.168.130.7 |

## node1~node4:

四台机器的hosts文件都配上域名映射,如下:

```
vi /etc/hosts
192.168.130.3 node1
192.168.130.4 node2
192.168.130.6 node3
192.168.130.7 node4
```

## 设置ssh免密

```
node1: 
	scp /root/.ssh/id_dsa.pub  node2:/root/.ssh/node1.pub
	scp /root/.ssh/id_dsa.pub  node3:/root/.ssh/node1.pub
	scp /root/.ssh/id_dsa.pub  node4:/root/.ssh/node1.pub
node2:
	cd ~/.ssh
	cat node01.pub >> authorized_keys
node3:
	cd ~/.ssh
	cat node01.pub >> authorized_keys
node4:
	cd ~/.ssh
	cat node01.pub >> authorized_keys
```

## 调整hadoop 配置

根据角色规划，重新设置 node1中 $HADOOP_HOME/etc/hadoop目录下的配置文件

```
vi core-site.xml
# 因为nameNode 我们还是放在原来的node1上，所以不用改啥
```

```
vi hdfs-site.xml
# secondNameNode 改到node2
# 几个角色的路径中local改成full
<configuration>
        <property>
                <name>dfs.replication</name>
                <value>2</value>
        </property>
        <property>
                <name>dfs.namenode.name.dir</name>
                <value>/var/bigdata/hadoop/full/dfs/name</value>
        </property>
        <property>
                <name>dfs.datanode.data.dir</name>
                <value>/var/bigdata/hadoop/full/dfs/data</value>
        </property>
        <property>
                <name>dfs.namenode.secondary.http-address</name>
                <value>node2:50090</value>
        </property>
        <property>
                <name>dfs.namenode.checkpoint.dir</name>
                <value>/var/bigdata/hadoop/full/dfs/secondary</value>
        </property>

</configuration>


```

```
vi slaves
node2
node3
node4
```

## 分发hadoop程序目录

```
cd /opt
	scp -r ./bigdata/  node2:`pwd`
	scp -r ./bigdata/  node3:`pwd`
	scp -r ./bigdata/  node4:`pwd`
```

## 格式化启动

```
hdfs namenode -format
start-dfs.sh
```

看已经初始化好了full目录，此次真分布式集群的nameNode数据就存在这 full/dfs/name

```
[root@node1 local]# cd /var/bigdata/hadoop/
[root@node1 hadoop]# ll
total 0
drwxr-xr-x. 3 root root 17 Apr 10 03:57 full
drwxr-xr-x. 3 root root 17 Apr  5 20:12 local
[root@node1 hadoop]#
```

```
#此时开始启动真正的分布式文件系统，在各个节点的/var/bigdata/hadoop/full/dfs/下创建的各个角色自己的目录，如 data,secondNameNode
[root@node1 dfs]# start-dfs.sh
Starting namenodes on [node1]
node1: starting namenode, logging to /opt/bigdata/hadoop-2.6.5/logs/hadoop-root-namenode-node1.out
node3: starting datanode, logging to /opt/bigdata/hadoop-2.6.5/logs/hadoop-root-datanode-node3.out
node2: starting datanode, logging to /opt/bigdata/hadoop-2.6.5/logs/hadoop-root-datanode-node2.out
node4: starting datanode, logging to /opt/bigdata/hadoop-2.6.5/logs/hadoop-root-datanode-node4.out
Starting secondary namenodes [node2]
node2: starting secondarynamenode, logging to /opt/bigdata/hadoop-2.6.5/logs/hadoop-root-secondarynamenode-node2.out

```

## web后台

可以看到hdfs完全分布式已经部署成功 http://127.0.0.1:50070/

（本机50070端口转发到虚拟机node1:50070）

![完全分布式hdfs](https://s2.loli.net/2022/04/10/2jzNXmADonpLe7R.png)

