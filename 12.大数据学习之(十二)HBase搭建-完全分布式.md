# HBase快速上手

## 前言

[Hbase](https://hbase.apache.org/) 是[Hadoop](https://hadoop.apache.org/)数据库，一种分布式、可扩展的大数据存储。

关键词：

- 实时nosql
- 半结构化，非结构化
- 线性可扩展
- 高效写入，高可用
- 自动故障转移
- 读取灵活性差
- 无事务处理

## 一、完全分布式安装

基于之前node1,node2,node3,node4 上的hadoop，hive环境继续搭建hbase。

### 1. 解压 hbase包

```shell
#node1
tar -zxvf hbase-2.0.5-bin.tar.gz  -C /opt/bigdata/
```

###  2.配置环境变量

```
#node1~node4都要
vi /etc/profile
#添加HBASE_HOME环境变量
export HBASE_HOME=/opt/bigdata/hbase-2.0.5/
export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$HIVE_HOME/bin:$HBASE_HOME/bin
#
source /etc/profile

#基于ntp 同步node1~node4时间
```

### 3.分配机器

| host  | NN   | NN   | JNN  | SNN  | DN   | FC   | ZK   | RM   | NM   | H-cLI | h-Mt-sV | HMASTER | RegionSVR |
| ----- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ----- | ------- | ------- | --------- |
| node1 | *    |      | *    |      |      | *    |      |      |      | *     |         | 1       |           |
| node2 |      | *    | *    | *    | *    | *    | *    |      | *    |       | *       |         | 1         |
| node3 |      |      | *    |      | *    |      | *    | *    | *    |       |         |         | 1         |
| node4 |      |      |      |      | *    |      | *    | *    | *    | *     |         | 1       | 1         |

前置条件：已经部署完 hadoop、hive

选择node1，node4来作为HMaster，node2,3,4作为regionServer

```
#node1,node4做ssh免密登录，如下先做node4免密访问node1
[root@node4 ~]# ssh-keygen
[root@node4 ~]# ssh-copy-id -i .ssh/id_rsa.pub node1
#输入密码，即可。
#试验免密ssh node1，然后同样ssh-copy-id到 node2,node3,node4
[root@node4 ~]# ssh node1
Last login: Tue Aug 16 02:05:02 2022 from node4
#同理实现 node1免密访问node1,2,3,4

```

### 4.hbase-env.sh文件修改

**node1**进入到/opt/bigdata/hbase-2.0.5/conf目录中，在hbase-env.sh文件中添加JAVA_HOME

```
#node1
#设置JAVA的环境变量
JAVA_HOME=/usr/java/default
设置是否使用自己的zookeeper实例
HBASE_MANAGES_ZK=false
```

### 5.hbase-site.xml文件修改

**node1**进入到/opt/bigdata/hbase-2.0.5/conf目录中，在hbase-site.xml文件中添加hbase相关属性

注意,mycluster 是hdfs的集群名称

```
<configuration>
  <property>
    <name>hbase.rootdir</name>
    <value>hdfs://mycluster/hbase</value>
  </property>
  <property>
    <name>hbase.cluster.distributed</name>
    <value>true</value>
  </property>
  <property>
    <name>hbase.zookeeper.quorum</name>
    <value>node2,node3,node4</value>
  </property>
</configuration>
```

### 6.修改regionservers文件

**node1**修改regionservers文件，设置regionserver分布在哪几台节点

```
node2
node3
node4
```

### 7.创建backup-masters文件

**node1**如果需要配置Master的高可用，需要在conf目录下创建backup-masters文件，并添加如下内容：

```
node4
```

### 8.配置hdfs

**node1**拷贝hdfs-site.xml文件到conf目录下

```
cp /opt/bigdata/hadoop-2.6.5/etc/hadoop/hdfs-site.xml /opt/bigdata/hbase-2.0.5/conf
```

### 9.分发hbase到其他机器

**node1**分发配置好的hbase到 node2，node3，node4

```
scp -r hbase-2.0.5/ node2:`pwd`
scp -r hbase-2.0.5/ node3:`pwd`
scp -r hbase-2.0.5/ node4:`pwd`
```

### 10.启动hdfs, 访问hdfs webUI 确认。

http://node1:50070

### 11.启动hbase

```
#node1
start-hbase.sh
```

```
#各个节点，检查进程 
jps
#HMaster（node1,node4）,HRegion（node2,node3,node4）
```

### 12.hbaseCli 报错解决

**node1**在任意目录下运行hbase shell的命令，进入hbase的命令行进行相关操作。

```
#打开终端
[root@node1 ~]# hbase shell;
#如果hbase shell报错，是因为hadoop的一个jar包太旧冲突，需要把替换成 hive lib下的。
#检查是否jar包太旧：比如这个包就是旧的jline-0.9.94.jar
[root@node1 bigdata]# ll hadoop-2.6.5/share/hadoop/yarn/lib/jline*
-rw-r--r--. 1 hadoop root 87325 May 24  2017 hadoop-2.6.5/share/hadoop/yarn/lib/jline-0.9.94.jar
#用这个替换即可
/opt/bigdata/hive-2.3.7/lib/jline-2.12.jar
#重启hbase
stop-hbase.sh
start-hbase.sh
```

### 13.webUI:

打开 http://192.168.56.103:16010/master-status#baseStats, 即搭建成功。

## 二、cli基本操作

![hbase表1](D:\学习资料\大数据学习笔记\static\hbase表1.png)

### 进入cli

通过cli进行简单的curd。

```
#node1
hbase shell
```

### 建表create

```
#建表，必须指定列族,默认是 default namespace
create 'tb1', 'cf'
create 'tb2', 'cf1','cf2'
#查看表
list
```

### 查询表scan 要慎用/ get 单行

```
# 示例 
scan 'namespace:tablename'
# 获取表的一行记录
hbase(main):000:0>get 't1', 'r1'
```

✔基于 RowKey的单行查询；
✔基于RowKey的范围查询(前缀匹配，避免缩小范围)；
❌全表扫描查询

### 查看表信息

```
#查看列族描述
describe 'tb1'
//获取表的切片
hbase(main):000:0>get_splits 't1'

```

### 插入数据

java client demo  [如何查看HBase的HFile - 大墨垂杨 - 博客园 (cnblogs.com)](https://www.cnblogs.com/quchunhui/p/7611565.html)

```
#c1 表示列族加列名，即 'cf:name'
#如果有多个列，则需要写多个put语句
#模拟插入 rowKey:1的行数据name,age,gender
put 'default:tb1','1','cf:name','ZhangSan'
put 'default:tb1','1','cf:age','18'
put 'default:tb1','1','cf:gender','male'
#测试删掉某个单元格
delete 'tb1', '1','cf:name'
put 'default:tb1','1','cf:name','ZhangSan'

#模拟插入 rowKey:2的行数据name,age,gender,position
put 'default:tb1','2','cf:name','Lisi'
put 'default:tb1','2','cf:age','19'
put 'default:tb1','2','cf:gender','male'
put 'default:tb1','2','cf:position','squad leader'

#查询数据
scan 'tb1'
#查看行数据
get 'tb1', '1'
//统计表的记录条数
count 'test'
#内存刷入磁盘，不然默认64MB才会刷
flush 'tb1'
#从hdfs cli/ hdfs-webUI 查看hfile
[root@node2 ~]# hdfs dfs -ls -h /hbase/data/default/tb1/d66ae140a94ef98a060e4beb6acb8ff9/cf
Found 1 items
-rw-r--r--   2 root supergroup      5.0 K 2022-08-19 00:26 /hbase/data/default/tb1/d66ae140a94ef98a060e4beb6acb8ff9/cf/9cbf4b462c074d6bb8d27aa78c353925

```

### 查看hfile 

```
#hdfs单机local模式
hbase hfile -p -f file:///home/testuser/hbase/data/default/tb1/{regionID}/cf/{数字}

#分布式
[root@node2 ~]# hbase hfile -p -f hdfs://mycluster/hbase/data/default/tb1/d66ae140a94ef98a060e4beb6acb8ff9/cf/9cbf4b462c074d6bb8d27aa78c353925
```


![hbase表2](D:\学习资料\大数据学习笔记\static\hbase表2.png)

### 查看HLog

[Hbase 预写日志WAL处理源码分析之 LogCleaner_学亮编程手记的技术博客_51CTO博客](https://blog.51cto.com/zhangxueliang/4946063?abTest=51cto)

Hlog失效和Hlog删除的规则

> HLog失效：写入数据一旦从MemStore中刷新到磁盘，HLog（默认存储目录在/hbase/WALs下）就会自动把数据移动到 /hbase/oldWALs 目录下，此时并不会删除

> Hlog删除：Master启动时会启动一个线程，定期去检查oldWALs目录下的可删除文件进行删除，定期检查时间为 hbase.master.cleaner.interval ，默认是1分钟 ，删除条件有两个：

### 删除数据/表

  ```
//删除表的指定rowkey的某个列数据
delete 'tb1', '2', 'cf:name'
//删除表的指定rowkey某个版本
hbase(main):000:0>delete 't1', 'r1', 'c1', ts1
//删除表的某一个列的所有值
hbase(main):000:0>deleteall 't1', 'r1', 'c1'

  ```

```
#删除表：先禁用才能删除
disable 'ns1:tb1'
drop 'ns1:tb1'
```

### WAL

直接往内存写数据，一般都需要wal来避免数据丢失，HLog同步/异步 数据可靠性。

### Compaction文件合并

```
#minor 3-10个文件合并
#major 当前目录所有文件合并

```

## 三、了解架构

![hbase架构图](D:\学习资料\大数据学习笔记\static\hbase架构图.png)

![hbase架构图3](D:\学习资料\大数据学习笔记\static\hbase架构图3.png)

![hbase架构图2](D:\学习资料\大数据学习笔记\static\hbase架构图2.png)

### 	角色：

#### 	（1）Client

​			1、包含访问HBase的接口并维护cache来加快对HBase的访问。

#### 	（2）Zookeeper

​			1、保证任何时候，集群中只有一个活跃master

​			2、存储所有region的寻址入口

​			3、实时监控region server的上线和下线信息，并实时通知master

​			4、存储HBase的schema和table元数据

#### 	（3）Master

​			1、为region server分配region

​			2、负责region server的负载均衡

​			3、发现失效的region server并重新分配其上的region

​			4、管理用户对table的增删改操作

#### 	（4）RegionServer

​			1、region server维护region，处理对这些region的IO请求

​			2、region server负责切分在运行过程中变得过大的region

### 	regionserver

#### 	（1）region

​			1、HBase自动把表水平划分成多个区域(region)，每个region会保存一个表里某段连续的数据

​			2、每个表一开始只有一个region，随着数据不断插入表，region不断增大，当增大到一个阈值的时候，region就会等分会两个新的region（裂变）

​			3、当table中的行不断增多，就会有越来越多的region。这样一张完整的表被保存在多个Regionserver 上。

#### 	（2）Memstore与storefile

​			1、一个region由多个store组成，一个store对应一个CF（列族）

​			2、store包括位于内存中的memstore和位于磁盘的storefile写操作先写入memstore，当memstore中的数据达到某个阈值，hregionserver会启动flashcache进程写入storefile，每次写入形成单独的一个storefile

​			3、当storefile文件的数量增长到一定阈值后，系统会进行合并（minor、major ），在合并过程中会进行版本合并和删除工作（majar），形成更大的storefile

​			4、当一个region所有storefile的大小和数量超过一定阈值后，会把当前的region分割为两个，并由hmaster分配到相应的regionserver服务器，实现负载均衡

​			5、客户端检索数据，先在memstore找，找不到去blockcache，找不到再找storefile

#### 	注意问题：

​			1、HRegion是HBase中分布式存储和负载均衡的最小单元。最小单元就表示不同的HRegion可以分布在不同的 HRegion server上。

​			2、HRegion由一个或者多个Store组成，每个store保存一个columns family。

​			3、每个Strore又由一个memStore和0至多个StoreFile组成。如图：StoreFile以HFile格式保存在HDFS上。

## 四、读写流程

### 	（1）读流程

​		1、客户端从zookeeper中获取meta表所在的regionserver节点信息

​		2、客户端访问meta表所在的regionserver节点，获取到region所在的regionserver信息

​		3、客户端访问具体的region所在的regionserver，找到对应的region及store

​		4、首先从memstore中读取数据，如果读取到了那么直接将数据返回，如果没有，则去blockcache读取数据

​		5、如果blockcache中读取到数据，则直接返回数据给客户端，如果读取不到，则遍历storefile文件，查找数据

​		6、如果从storefile中读取不到数据，则返回客户端为空，如果读取到数据，那么需要将数据先缓存到blockcache中（方便下一次读取），然后再将数据返回给客户端。

​		**7、blockcache是内存空间，如果缓存的数据比较多，满了之后会采用LRU策略，将比较老的数据进行删除。**

### 	（2）写流程

​		1、客户端从zookeeper中获取meta表所在的regionserver节点信息

​		2、客户端访问meta表所在的regionserver节点，获取到region所在的regionserver信息

​		3、客户端访问具体的region所在的regionserver，找到对应的region及store

​		4、开始写数据，写数据的时候会先想hlog中写一份数据（方便memstore中数据丢失后能够根据hlog恢复数据，向hlog中写数据的时候也是优先写入内存，后台会有一个线程定期异步刷写数据到hdfs，如果hlog的数据也写入失败，那么数据就会发生丢失）

​		5、hlog写数据完成之后，会先将数据写入到memstore，memstore默认大小是64M，当memstore满了之后会进行统一的溢写操作，将memstore中的数据持久化到hdfs中，

​		**6、频繁的溢写会导致产生很多的小文件，因此会进行文件的合并，文件在合并的时候有两种方式，minor和major，minor表示小范围文件的合并，major表示将所有的storefile文件都合并成一个。**

## 五、LSM的设计思想和原理

LSM树的设计思想非常简单：将对**数据的修改增量保持在内存中**，达到指定的大小限制后将这些修改**操作批量写入磁盘**，不过**读取**的时候稍微麻烦，需要**合并磁盘中历史数据和内存中最近修改操作**，所以**写入性能大大提升**，读取时可能需要先看是否命中内存，否则需要访问较多的磁盘文件。极端的说，**基于LSM树实现的HBase的写性能比Mysql高了一个数量级，读性能低了一个数量级**。

LSM树原理把一棵大树拆分成N棵小树，它首先写入内存中，随着小树越来越大，内存中的小树会flush到磁盘中，磁盘中的树定期可以做merge操作，合并成一棵大树，以优化读性能。

在hbase中LSM的应用流程对应说下：

​		1、因为小树先写到内存中，为了防止内存数据丢失，写内存的同时需要暂时持久化到磁盘，对应了HBase的MemStore和HLog

​		2、MemStore上的树达到一定大小之后，需要flush到HRegion磁盘中（一般是Hadoop DataNode），这样MemStore就变成了DataNode上的磁盘文件StoreFile，定期HRegionServer对DataNode的数据做merge操作，彻底删除无效空间，多棵小树在这个时机合并成大树，来增强读性能。

![树合并](D:\学习资料\大数据学习笔记\static\树合并.png)



## 六、MSLAB - MemStore内存优化

[HBASE MSLAB对memstore的GC优化](https://blog.csdn.net/weixin_40954192/article/details/106892432)

## 七、优化

#### rowKey的设计！！！

例如: rowkey=哈希（主键<递增的id\手机号码等>）+Long.Max_Value - timestamp

参考连接：[(86条消息) Hbase Rowkey CF 架构 概述 预分区及Rowkey设计 学习笔记_小鹅鹅的博客-CSDN博客](https://blog.csdn.net/asd136912/article/details/86310966)

#### 预分区

#### 列族的设计

#### blockcache调优

#### 建表时指定Time to Live

#### 文件合并优化（系统资源消耗）

https://blog.csdn.net/zc19921215/article/details/105931033/

#### 批量读写

#### WAL同步/异步

#### region是否太少

#### 请求是否均衡

#### scan缓存是否合理

#### 请求是否可以指定列族/列

#### HFile文件是否太多

#### 数据尽量本地读写

#### JVM调优

## 七、hbase架构参考资料

hbase学习

#### LSM B+ 数对比

http://www.360doc.com/content/21/0524/08/65762893_978694803.shtml

#### HFile 文件格式

https://blog.csdn.net/x541211190/article/details/108387698
https://blog.csdn.net/weixin_39682511/article/details/111012495
好文https://blog.csdn.net/xiaohu21/article/details/108344591
https://www.likecs.com/show-204823334.html
https://developer.aliyun.com/article/752817

#### hbase 介绍，逻辑/物理 视图，RDBM对比

把主键哈希后当成rowkey的头部
https://blog.csdn.net/asd136912/article/details/86310966

#### hbase 分裂

region split会消耗宝贵的集群I/O资源
https://blog.csdn.net/ThreeAspects/article/details/104310627
https://blog.csdn.net/asd136912/article/details/101168177

#### hbase分区

HBaseWebUI
https://blog.csdn.net/liu16659/article/details/80738742

#### 优化-分裂与合并

https://www.modb.pro/db/188561
https://blog.csdn.net/lightupworld/article/details/106491383
好文 https://blog.csdn.net/zc19921215/article/details/105931033/

#### 读写流程分析

读缓存有误https://blog.csdn.net/zc19921215/article/details/86658310

## 八、hbase整合 hive

官方文档[HBaseIntegration - Apache Hive - Apache Software Foundation](https://cwiki.apache.org/confluence/display/Hive/HBaseIntegration)

简述：把hive-site.xml配上zookeer  即可。
原理：存储引擎换成 hbaseStorageHandler，hive表建立与hbase表的映射，列映射

注意，需要启动整个hive集群(hdfs,yarn,hive), 以及hbase

### 1. 通过hive sql 内部表 来写入、查询hbase

```
#node2
hive --service metastore

#node4
cd /opt/bigdata/hive-2.3.4/conf
#备份hive-site.xml，添加如下配置
vi hive-site.xml
<property>
	<name>hbase.zookeeper.quorum</name>
	<value>node2,node3,node4</value>
</property>

#node4启动hive,这样 hive就可以连接到hbase
hive
#此时hive原来的已经存在的表，跟hbase并没有建立映射关系
#需要映射hive表列 到 hbase的列族:列
#在hive建表
CREATE TABLE hbase_table_1(key int, value string) 
STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'
WITH SERDEPROPERTIES ("hbase.columns.mapping" = ":key,cf1:val")
TBLPROPERTIES ("hbase.table.name" = "xyz", "hbase.mapred.output.outputtable" = "xyz");  
#查看hive种中是否成功创建hbase_table_1
show tables;
#切到 node1,hbase cli 查看是否创建对应的hbase表 xyz； ps: 可以建hive外部表关联已经存在的hbase表
list
#node1 ，  hbase cli确定hbase表 xyz 是空的
scan 'xyz'

#node4，确定hive表hbase_table_1 是空的
select * from hbase_table_1;

#查看hdfs-webui里表对应的目录名 是hive表名还是hbase表名
#hive表名目录存在，但是数据不会往里面存，下面插入数据验证
/usr/hive_remote/warehourse/hbase_table_1
#node4 hive cli执行数据插入语句
insert into hbase_table_1 values(1,'zhangsan');
#node4查看hive表数据
select * from hbase_table_1;

#node1 查看hbase表数据
scan 'xyz';
#查看hdfs-webui里表对应的目录名
#hive表名目录存在，但是没数据
/usr/hive_remote/warehourse/hbase_table_1

#查看hdfs-webui里表对应的目录名
#node1 hbase-cli
fllush 'xyz'
#hbase表名目录存在，有数据。---验证完毕。
/hbase/data/default/xyz/xxxx(storeFileName)xxxx/cf1


#hbase插入数据试验
#node1 插入如
put 'xyz','2','cf1:name','lisi'
scan 'xyz'
#node4 hive cli查数据,，此时查不到 rowKey '2'的数据
#原因：因为插入hbase列名是 name，跟hive表的映射不一样。
select * from hbase_table_1;

#node1 插入如
put 'xyz','2','cf1:val','lisi'

#node4 hive cli查数据,，此时可以查到 rowKey '2'的数据
select * from hbase_table_1;



```

### 2. 通过hive sql 外部表 来写入、查询hbase

hbase已经有表，此时建立hive外部表进行关联，**更常见**。

 ```
#node1
create 'some','cf1'

#node4
CREATE EXTERNAL TABLE hbase_table_2(key int, value string) 
STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'
WITH SERDEPROPERTIES ("hbase.columns.mapping" = "cf1:val")
TBLPROPERTIES("hbase.table.name" = "some", "hbase.mapred.output.outputtable" = "some");

#node1
put 'some','1','cf1:val','zhangsan'

#node4 可以查到
select * from hbase_table_2;

 ```





