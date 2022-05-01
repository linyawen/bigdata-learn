



# 一、mapreduce 计算模型分析

map 和reduce的原理，把下面两张图看明白就基本理解了。

ps: 这两张图没有画出 **combiner**，combiner基本等同于reduce，只是执行的地方/时机不同。

好处：在map节点提前进行汇总统计，避免数据 汇总到 reduce的大量IO，也利用了map的计算资源。

注意：使用combiner提前计算是否对业务有影响。



![mapreduce图解01](https://s2.loli.net/2022/04/25/1Mkmt76yi4nDNKv.png)

下面这个图更详细且严谨

![mapreduce图解02](https://s2.loli.net/2022/04/25/xtPs93IJFH8nmBY.png)

# 二、yarn架构图

在hadoop分布式集群中，yarn负责资源管理，以及任务的调度。支持hadoop以及spark等计算框架。

ps: 

- MapReduce jar包（计算）怎么向数据移动，jar包的执行调度/监控靠的是 yarn。
- ResourceManager 支持高可用。
- 编写mapReduce程序时，最好预估内存用量（文件块大小，业务计算逻辑等因素），适当调整yarn分配container的资源（默认1C1G）

![yarn图解01](https://s2.loli.net/2022/04/25/IDLd5itC3m9wQ81.png)



# 三、yarn搭建

## 1.角色规划

在之前ha集群的架构基础上继续部署:

- RM: resourceManager，和NameNode错开，选Node3,Node4进行高可用，RM内部实现了高可用，不用像NameNode依赖zkfc。
- NM: nodeManager，**每个dataNode节点都要部署NM**，因为是由NM来在本机启动container执行mapreduce。

| host  | NN   | NN   | JNN  | SNN  | DN   | FC   | ZK   | RM   | NM   |
| ----- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| node1 | *    |      | *    |      |      | *    |      |      |      |
| node2 |      | *    | *    | *    | *    | *    | *    |      | 1    |
| node3 |      |      | *    |      | *    |      | *    | 1    | 1    |
| node4 |      |      |      |      | *    |      | *    | 1    | 1    |

## 2.部署node1

hdfs等所有都用root启动。

```
cd /opt/bigdata/hadoop-2.6.5/etc/hadoop
cp mapred-site.xml.template mapred-site.xml
vi mapred-site.xml

```

### vi mapred-site.xml

ps: <value>local</value> 即单进程运行mapreduce，适合调试使用。

```
<property>
<name>mapreduce.framework.name</name>
<value>yarn</value>
</property>
```

### vi yarn-site.xml

```
<property>
<name>yarn.nodemanager.aux-services</name>
<value>mapreduce_shuffle</value>
</property>

<property>
<name>yarn.resourcemanager.ha.enabled</name>
<value>true</value>
</property>
<property>
<name>yarn.resourcemanager.zk-address</name>
<value>node2:2181,node3:2181,node4:2181</value>
</property>

<property>
<name>yarn.resourcemanager.cluster-id</name>
<value>mashibing</value>
</property>

<property>
<name>yarn.resourcemanager.ha.rm-ids</name>
<value>rm1,rm2</value>
</property>
<property>
<name>yarn.resourcemanager.hostname.rm1</name>
<value>node3</value>
</property>
<property>
<name>yarn.resourcemanager.hostname.rm2</name>
<value>node4</value>
</property>
```

### 分发到node2,3,4

```
scp mapred-site.xml yarn-site.xml  node2:`pwd`
scp mapred-site.xml yarn-site.xml  node3:`pwd`
scp mapred-site.xml yarn-site.xml  node4:`pwd`
```

### NodeManager配置

不用配置 ，就是之前 配置 datanode的slaves文件。

### 启动yarn

#### 0.前提：启动好ZK, JN,HDFS

```
#node2,3,4
zkServer.sh start

#在node1，node2,node3 依次执行
hadoop-daemon.sh start journalnode

#node1
start-dfs.sh
```

#### 1.启动NodeManager

**问题**：发现resourceManager并没有按照我们的配置在node3，4启动，而且也没有如同log显示那样启动在node1上。

所以此时**只正确启动了NodeManager.**

```
[root@node1 hadoop]# start-yarn.sh 
starting yarn daemons
starting resourcemanager, logging to /opt/bigdata/hadoop-2.6.5/logs/yarn-root-resourcemanager-node1.out
node2: starting nodemanager, logging to /opt/bigdata/hadoop-2.6.5/logs/yarn-root-nodemanager-node2.out
node4: starting nodemanager, logging to /opt/bigdata/hadoop-2.6.5/logs/yarn-root-nodemanager-node4.out
node3: starting nodemanager, logging to /opt/bigdata/hadoop-2.6.5/logs/yarn-root-nodemanager-node3.out
```

#### 2.**启动resourceManager**

**改正**：手工在node3，node4上**启动resourceManager.**

```
node3：
yarn-daemon.sh start resourcemanager
node4:
yarn-daemon.sh start resourcemanager
```

#### 3.检查zk

resourceManager rm1已经抢锁成功

```
[zk: localhost:2181(CONNECTED) 0] ls /
[zookeeper, hadoop-ha]
[zk: localhost:2181(CONNECTED) 1] ls /
[zookeeper, yarn-leader-election, hadoop-ha]
[zk: localhost:2181(CONNECTED) 2]  ls /yarn-leader-election
[mashibing]
[zk: localhost:2181(CONNECTED) 3] ls /yarn-leader-election/mashibing
[ActiveBreadCrumb, ActiveStandbyElectorLock]
[zk: localhost:2181(CONNECTED) 4] get /yarn-leader-election/mashibing/ActiveStandbyElectorLock 

	mashibingrm1
cZxid = 0x40000000c
ctime = Fri Apr 29 00:40:10 CST 2022
mZxid = 0x40000000c
mtime = Fri Apr 29 00:40:10 CST 2022
pZxid = 0x40000000c
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x38071015ea60000
dataLength = 16
numChildren = 0
```

#### 4. Yarn 控制台

浏览器打开node3的resourceManger端口 http://node3:8088

![yarn_resourceManger_0](https://s2.loli.net/2022/04/29/cEl8RGZkAeLn9Sp.png)

注意：Memory Total，VCores Total显示有问题，需要改配置。

# 四、在yarn上运行mapreduce

实战：MR ON YARN 的运行方式：

```
#创建输入数据目录
hdfs dfs -mkdir -p   /data/wc/input 
#上传文件
hdfs dfs -D dfs.blocksize=1048576  -put data.txt  /data/wc/input
#执行wordcount demo
cd  $HADOOP_HOME/share/hadoop/mapreduce
hadoop jar  hadoop-mapreduce-examples-2.6.5.jar   wordcount   /data/wc/input   /data/wc/output

#查看应用执行结果的两种方式
1)webui:
2)cli:
#cli方式查看
hdfs dfs -ls /data/wc/output
-rw-r--r--   2 root supergroup          0 2019-06-22 11:37 /data/wc/output/_SUCCESS  //标志成功的文件
-rw-r--r--   2 root supergroup     788922 2019-06-22 11:37 /data/wc/output/part-r-00000  //数据文件
#reduce的输出文件part-r-00000，如果没有reduce只有map，那文件名如part-m-00000
#两种文件r/m :  map+reduce   r   / map  m
#查看结果文件内容
hdfs dfs -cat  /data/wc/output/part-r-00000
#保存结果到本地磁盘
hdfs dfs -get  /data/wc/output/part-r-00000  ./
```



# 五、在idea 开发mapreduce

1. 可以拿官方wordCount demo来进行改造。

2. maven引入 跟集群一致的hadoop-client。

3. 把集群的core-site.xml, hdfs-site.xml,mapred-site.xml,yarn-site.xml下载到本地 resources。

4. 初始化job：

   1.  指定本应用的数据来源，输出源，一般是dfs目录。
   2. 指定读取本地配置 resources里面的xml. 
   3. 指定MapperClass,ReducerClass。
   4. 指定mapper排序逻辑，优化效率。注意：字典序，数值序。

5. 实现MapperClass,ReducerClass业务逻辑.

   1. 注意大数据场景下程序编写的优化.

      比如:循环 new 变量,可能会被大量调用gc .

      解决:写在类静态变量,通过hadoop自带的类型会序列化,不会有引用问题.

   2. 最好用combiner把聚合计算前置在mapper处理,提升效率.

   

# 六、配置优先级

注意:  有几个地方都存在 xxxx-site.xml 配置,注意优先级如下降序:

1. maven项目中 java 代码设置的配置值.
2. maven项目中resources的配置.
3. 服务器$HADOOP_HOME/etc里的配置.
4. hadoop框架.jar包内置的默认配置.(一般不用)

# 七、故障记录

## 1.集群里缺失节点。

### 问题描述

提交几次mapReduce以后，yarn里一台少了一台Node4，重启集群后node4有出现在 hdfs里，但是yarn里还是没有。

### 分析

hadoop-root-datanode-node4.log 显示异常如下：

```
 org.apache.hadoop.hdfs.server.datanode.DataNode: RECEIVED SIGNAL 15: S
```

**应该是dataNode被集群杀掉了。**

偶然观察到node4 磁盘只剩下500MB（测试的虚拟机空间很小）。

尝试删掉点东西，空间有800MB了，刷新下yarn，节点出现了。。。

观察到 yarn-root-nodemanager-node4.log 里有磁盘相关的信息  Disk(s) turned good

```
2022-05-01 19:24:18,290 INFO org.apache.hadoop.yarn.server.nodemanager.DirectoryCollection: Directory /tmp/hadoop-root/nm-local-dir passed disk check, adding to list of valid directories.
2022-05-01 19:24:18,290 INFO org.apache.hadoop.yarn.server.nodemanager.DirectoryCollection: Directory /opt/bigdata/hadoop-2.6.5/logs/userlogs passed disk check, adding to list of valid directories.
2022-05-01 19:24:18,290 INFO org.apache.hadoop.yarn.server.nodemanager.LocalDirsHandlerService: Disk(s) turned good: 1/1 local-dirs are good: /tmp/hadoop-root/nm-local-dir; 1/1 log-dirs are good: /opt/bigdata/hadoop-2.6.5/logs/userlogs
```

应该是我腾出磁盘盘空间后，NodeManager健康检查变成通过了，就加回集群里了，那应该前面会有磁盘检查有关的错误日志，果然找到如下:

```
2022-05-01 19:18:18,166 WARN org.apache.hadoop.yarn.server.nodemanager.DirectoryCollection: Directory /tmp/hadoop-root/nm-local-dir error, used space above threshold of 90.0%, removing from list of valid directories
2022-05-01 19:18:18,166 WARN org.apache.hadoop.yarn.server.nodemanager.DirectoryCollection: Directory /opt/bigdata/hadoop-2.6.5/logs/userlogs error, used space above threshold of 90.0%, removing from list of valid directories
2022-05-01 19:18:18,166 INFO org.apache.hadoop.yarn.server.nodemanager.LocalDirsHandlerService: Disk(s) failed: 1/1 local-dirs are bad: /tmp/hadoop-root/nm-local-dir; 1/1 log-dirs are bad: /opt/bigdata/hadoop-2.6.5/logs/userlogs
2022-05-01 19:18:18,167 ERROR org.apache.hadoop.yarn.server.nodemanager.LocalDirsHandlerService: Most of the disks failed. 1/1 local-dirs are bad: /tmp/hadoop-root/nm-local-dir; 1/1 log-dirs are bad: /opt/bigdata/hadoop-2.6.5/logs/userlogs

```

### **结论**

 保持机器的磁盘，内存，cpu 资源足够，如果突然机器被提出集群，优先考虑健康导致。

