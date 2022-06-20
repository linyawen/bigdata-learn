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

# Hive索引

# Hive优化

类似mysql优化，mysql优化主要是理解B+树，磁盘IO。

hive底层是把sql转换成hadoop执行，所以要从Hadoop的执行过程来思考优化方式。

**思考**：hive支持把计算引擎切换成 spark，那是不是同样的语句的优化方式就变了？

# Hive权限管理