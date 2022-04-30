



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
cd  $HADOOP_HOME
cd share/hadoop/mapreduce
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

 



