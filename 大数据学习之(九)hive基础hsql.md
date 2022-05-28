



# 1.创建hive表 create table psn

**注意事项：**

1. 指定分隔符
2. 最好用外部表

```sql
-- 建测试库
create database test;

create table psn
(
id int,
name string,
likes array<string>,
address map<string,string>
)
row format delimited
fields terminated by ','
collection items terminated by '-'
map keys terminated by ':';

```

# 2.查看表信息 desc formatted

```
hive>  desc formatted psn;
OK
# col_name            	data_type           	comment             
	 	 
id                  	int                 	                    
name                	string              	                    
likes               	array<string>       	                    
address             	map<string,string>  	                    
	 	 
# Detailed Table Information	 	 
Database:           	default             	 
Owner:              	root                	 
CreateTime:         	Tue May 24 00:57:38 CST 2022	 
LastAccessTime:     	UNKNOWN             	 
Retention:          	0                   	 
Location:           	hdfs://mycluster/user/hive_remote/warehouse/psn	 
Table Type:         	MANAGED_TABLE       	 
Table Parameters:	 	 
	COLUMN_STATS_ACCURATE	{\"BASIC_STATS\":\"true\"}
	numFiles            	0                   
	numRows             	0                   
	rawDataSize         	0                   
	totalSize           	0                   
	transient_lastDdlTime	1653325058          
	 	 
# Storage Information	 	 
SerDe Library:      	org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe	 
InputFormat:        	org.apache.hadoop.mapred.TextInputFormat	 
OutputFormat:       	org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat	 
Compressed:         	No                  	 
Num Buckets:        	-1                  	 
Bucket Columns:     	[]                  	 
Sort Columns:       	[]                  	 
Storage Desc Params:	 	 
	colelction.delim    	-                   
	field.delim         	,                   
	mapkey.delim        	:                   
	serialization.format	,                   
Time taken: 0.252 seconds, Fetched: 36 row(s)

```

# 3.load本地文件到表里

直接insert 太慢，一条要几十秒。

一般用标准格式化的文件直接load到hive表里**，hive不会对文件做任何处理**。

```
#node4，
#local inpath 表示本地文件，省略local则表示直接从hdfs路径加载。
#从hdfs路径加载会更快。不用像local这样还要上传。
load data local inpath '/root/data/data' into table psn;
```

可以看到，耗时才4秒，比insert 快多了

```
hive> load data local inpath '/root/data/data' into table psn;
Loading data to table default.psn
OK
Time taken: 3.933 seconds

```

# 4.从表里读数据

```
hive> select * from psn;
OK
1	小明1	["王者","吃鸡"]	{"思明":"软二","集美":"软三"}
2	小明2	["吃鸡"]	{"思明":"软二","集美":"软三"}
3	小明3	["吃鸡","竞速"]	{"思明":"软二","集美":"软三"}
4	小明4	["竞速"]	{"思明":"软二","集美":"软三"}
5	小明5	["王者","吃鸡","竞速"]	{"思明":"软二","集美":"软三"}
6	小明6	["王者","连连看"]	{"集美":"软三"}
7	小明7	["贪吃蛇","王者"]	{"思明":"观音山","集美":"软三"}
8	小明8	["lol","吃鸡"]	{"集美":"软三"}
9	小明9	["dota","吃鸡"]	{"思明":"软二"}
10	小明10	["war3","吃鸡"]	{"思明":"软二"}
Time taken: 1.645 seconds, Fetched: 10 row(s)

```

# 5.直接hdfs dfs -put到表里

```
#node4，再来一个data2 文件，上传到表同名目录
hdfs dfs -put data2 /user/hive_remote/warehouse/psn

```

再次搜索，能看到data2的数据

```
hive> select * from psn;
OK
1	小明1	["王者","吃鸡"]	{"思明":"软二","集美":"软三"}
2	小明2	["吃鸡"]	{"思明":"软二","集美":"软三"}
3	小明3	["吃鸡","竞速"]	{"思明":"软二","集美":"软三"}
4	小明4	["竞速"]	{"思明":"软二","集美":"软三"}
5	小明5	["王者","吃鸡","竞速"]	{"思明":"软二","集美":"软三"}
6	小明6	["王者","连连看"]	{"集美":"软三"}
7	小明7	["贪吃蛇","王者"]	{"思明":"观音山","集美":"软三"}
8	小明8	["lol","吃鸡"]	{"集美":"软三"}
9	小明9	["dota","吃鸡"]	{"思明":"软二"}
10	小明10	["war3","吃鸡"]	{"思明":"软二"}
11	大明1	["王者","吃鸡"]	{"思明":"软二","集美":"软三"}
12	大明2	["吃鸡"]	{"思明":"软二","集美":"软三"}
13	大明3	["吃鸡","竞速"]	{"思明":"软二","集美":"软三"}
14	大明4	["竞速"]	{"思明":"软二","集美":"软三"}
15	大明5	["王者","吃鸡","竞速"]	{"思明":"软二","集美":"软三"}
16	大明6	["王者","连连看"]	{"集美":"软三"}
17	大明7	["贪吃蛇","王者"]	{"思明":"观音山","集美":"软三"}
18	大明8	["lol","吃鸡"]	{"集美":"软三"}
19	大明9	["dota","吃鸡"]	{"思明":"软二"}
20	大明10	["war3","吃鸡"]	{"思明":"软二"}
Time taken: 0.318 seconds, Fetched: 20 row(s)
```

**总结：**

1.上传文件到hive的hdfs目录时不会做任何检查处理，而是读时检查。（ps:  反之，mysql是写时检查）

2.hive sql 读数据时，如果发现目录下有文件格式不对，那对应行会显示几个null列。

# 6.表类型,内部表，外部表

1. hive默认创建的是内部表。
2. 外部表，建表时参数指定 external  ,location hdfs_path .

区别：

1，建表时，内部表存储在hive的默认hdfs目录中。

外部表时存在指定的location path中。

2，删表时，  内部表会把hdfs数据和mysql元数据全部删除。

外部表不会删除hdfs数据。

应用场景：

内部表：先创建表再添加数据，

外部表：无所谓先后，甚至可以先收集数据再创建表.

# 7.DDL 表分区

```
create table psn2
(
id int,
name string,
likes array<string>,
address map<string,string>
)
partitioned by (gender string)
row format delimited
fields terminated by ','
collection items terminated by '-'
map keys terminated by ':';


```

###    7.1 load数据时要指定属于哪个分区 

这个命令会先创建分区目录再加载数据,并且为分区列赋值  


```
load data local inpath '/root/data/data' into table psn2 partition(gender='boy');
```

###    7.2 多分区注意

```
partitioned by (gender string,age int)
```
多分区在load数据时，必须指定所有分区，不能只指定一个
```
load data local inpath '/root/data/data' into table psn2 partition(gender='boy',age=12);
```

### 7.3  分区列添加 

前两部都是在load数据时指定分区来生成目录。

也可以事先生成好，必须指定所有分区

```
alter table psn2 add partition(gender='girl',age=12);
```

### 7.4 分区删除 

分区删除，不需要指定所有分区

```
alert table psn2 drop partition(age=12)
```

会自动把 gender=boy/age=12,gender=girl/age=12 的分区目录都删了

### 7.5 todo动态分区 

根据数据行中某个属性来决定分区 TODO

### 7.6 修复分区

如果是手工在hdfs建好表、分区目录，再去hive createtable，此时 mysql里没有分区元数据，会导致查不数据，需要做如下修复分区：来补s充元数据。

```
msck repair table psn2;
```

# 8 DML 

## 8.1通过查询结果 填充子表

**实际情况下使用很频繁。**

```
from  TABLE_NAME1
insert override table sub_table1
select COLUMN1,COLUMN2
insert override table sub_table2
select COLUMN3
```

## 8.2通过查询结果存入本地文件目录

很少用

```
insert override local directory '/root/result' select * from TABLE_NAME
```

## 8.3 事务管理

**一般不用**

修改，删除动作必须保证事务开启，hdfs 文件只能append，不能修改，但是hive经过很多配置开启以后才能支持有限的修改，很多限制条件。

原理：hive把**整个文件块**复制下来，修改以后重新放回hdfs。



 

54:-00