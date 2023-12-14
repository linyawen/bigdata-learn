# Hive动态分区&分桶

## 一、分区

hdfs层面把不同分区的数据放在不同的目录，避免全量io。

```
--hive设置hive动态分区开启
	set hive.exec.dynamic.partition=true;
	默认：true
--hive的动态分区模式
	set hive.exec.dynamic.partition.mode=nostrict;
	默认：strict（至少有一个分区列是静态分区）
--每一个执行mr节点上，允许创建的动态分区的最大数量(100)
	set hive.exec.max.dynamic.partitions.pernode;
--所有执行mr节点上，允许创建的所有动态分区的最大数量(1000)	
	set hive.exec.max.dynamic.partitions;
--所有的mr job允许创建的文件的最大数量(100000)	
	set hive.exec.max.created.files;
```

### 单分区表示例

```
create table psn5
(
id int,
name string
)
partitioned by(gender string)
row format delimited
fields terminated by ',';
```

```
#insert语句携带分区
insert into psn5 PARTITION(gender="boy") values(1,'zhangsan');
```

多来几条数据,

注意：分区名用中文会有问题。

```
insert into psn5 PARTITION(gender="boy") values(2,'zhangsan2');
insert into psn5 PARTITION(gender="boy") values(3,'zhangsan3');
insert into psn5 PARTITION(gender="boy") values(4,'zhangsan4');
insert into psn5 PARTITION(gender="girl") values(5,'hanmeimei');

```

观察dfs分区目录如下：

```
hive> dfs -ls /user/hive_remote/warehouse/psn5
    > ;
Found 3 items
drwxr-xr-x   - root supergroup          0 2022-06-20 00:23 /user/hive_remote/warehouse/psn5/gender=boy
drwxr-xr-x   - root supergroup          0 2022-06-20 00:23 /user/hive_remote/warehouse/psn5/gender=girl
drwxr-xr-x   - root supergroup          0 2022-06-19 23:59 /user/hive_remote/warehouse/psn5/gender=男
```

### 多分区表

```
--创建多分区表
	create table psn6
	(
	id int,
	name string,
	likes array<string>,
	address map<string,string>
	)
	partitioned by(gender string,age int)
	row format delimited
	fields terminated by ','
	collection items terminated by '-'
	map keys terminated by ':';	
/*
	注意：
		1、当创建完分区表之后，在保存数据的时候，会在hdfs目录中看到分区列会成为一个目录，以多级目录的形式			  存在
		2、当创建多分区表之后，插入数据的时候不可以只添加一个分区列，需要将所有的分区列都添加值
		3、多分区表在添加分区列的值得时候，与顺序无关，与分区表的分区列的名称相关，按照名称就行匹配
*/	
--给分区表添加分区列的值
	alter table table_name add partition(col_name=col_value)
--删除分区列的值
	alter table table_name drop partition(col_name=col_value)
/*
	注意:
		1、添加分区列的值的时候，如果定义的是多分区表，那么必须给所有的分区列都赋值
		2、删除分区列的值的时候，无论是单分区表还是多分区表，都可以将指定的分区进行删除
*/
```



## 二、分桶

和分区不同的是，Hive分桶表是对列值取hash值得方式，将不同数据放到**不同文件**中存储。

对于hive中每一个表、分区都可以进一步进行分桶

**主要用于抽样**。

# Hive视图

简化查询语句

# Hive索引

比较少用，每次数据增加后，需要手工构建索引。

# Hive的抓取策略

简单的操作不再需要转换成MapReduce，例如

- select 仅支持本表字段
- where仅对本表字段做条件过滤

# Hive优化

## （一）hadoop mr 优化。

类似mysql优化，mysql优化主要是理解B+树，磁盘IO。

hive底层是把sql转换成hadoop执行，所以要从Hadoop的执行过程来思考优化方式。

**思考**：hive支持把计算引擎切换成 spark，那是不是同样的语句的优化方式就变了？

- **Hive执行计划**

- ## MR的splitsize优化，

- ## 合理控制 map，reduce 在每台机器的数量。

- Hive join
  Hive 在多个表的join操作时尽可能多的使用相同的连接键，这样在转换MR任务时会转换成少的MR的任务。

- 手动Map join:在map端完成join操作。

- Map-Side聚合，Hive的某些SQL操作可以实现map端的聚合，类似于MR的combine操作

## （二）jvm优化

## （三）hdfs空间优化

### 空间优化 ：

- ### 文件压缩格式 snappy

- 合并小文件、

## （四） 存储方式优化

选择合适的文件格式，text一般用的比较少，一般企业里用**列式存储**等。

~~行式存储:textfile sequencefile~~

**列式存储**: orc ,**parquet**，重点 parquet,面向分析型业务的烈士存储格式。

优点:   

- 列式更省空间。

- 减小IO量。

- **spark能直接读取 parquet文件，**不需要经过转换。

-  **orc优点**： 可以压缩到 textfile接近5，6倍，但是

缺点：

- 不能直接看hdfs里的文件了，可能导致查询变慢，具体要试验对比。orc压缩率很高。
- 加载到表后不能再插入数据了，所以适合历史数据


## （五）高可用

keepalive或zookeeper，一般共zk

metaStore不用做高可用，mysql集群要做高可用，hive server2也要做高可用。

## （六）Hive并行模式

在SQL语句足够复杂的情况下，可能在一个SQL语句中包含多个子查询语句，且多个子查询语句之间没有任何依赖关系，此时，可以Hive运行的并行度

```sql
--设置Hive SQL的并行度
set hive.exec.parallel=true;
```

		注意：Hive的并行度并不是无限增加的，在一次SQL计算中，可以通过以下参数来设置并行的job的个数

```sql
--设置一次SQL计算允许并行执行的job个数的最大值
set hive.exec.parallel.thread.number
```



## （七）Hive严格模式

Hive中为了提高SQL语句的执行效率，可以设置严格模式，充分利用Hive的某些特点。

```sql
-- 设置Hive的严格模式
set hive.mapred.mode=strict;
```

注意：当设置严格模式之后，会有如下限制：

​				（1）对于分区表，必须添加where对于分区字段的条件过滤

​				（2）order by语句必须包含limit输出限制

​				（3）限制执行笛卡尔积的查询

## （八）Hive排序

​		在编写SQL语句的过程中，很多情况下需要对数据进行排序操作，Hive中支持多种排序操作适合不同的应用场景。

​		1、Order By - 对于查询结果做全排序，只允许有一个reduce处理
​			（当数据量较大时，应慎用。严格模式下，必须结合limit来使用）
​		2、Sort By - 对于单个reduce的数据进行排序
​		3、Distribute By - 分区排序，经常和Sort By结合使用
​		4、Cluster By - 相当于 Sort By + Distribute By
​			（Cluster By不能通过asc、desc的方式指定排序规则；
​				可通过 distribute by column sort by column asc|desc 的方式）

#  hiveserver2：

企业多用hiveserver2，**优点**：

1. #### 应用端不需要部署hadoop，hive cli。

2. 不用直接将hdfs 和 metastore暴露给用户用户。

3. 可以做高可用，解决应用端并发和负载问题。

4. jdbc跨语言。



 

# Hive权限管理

