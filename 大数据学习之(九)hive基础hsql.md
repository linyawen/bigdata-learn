



# 1.创建hive表 

**注意事项：**

1. 指定分隔符

2. 最好用外部表

3. 默认分隔符为 ^A.

   一般情况下看不见这个符号，除非把hdfs文件下载下来，用cat -A 命令查看

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
#看列信息
desc tablename;
#列信息，分隔符信息等
desc formatted tablename;
```

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

# 5.hdfs dfs -put数据到表里

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

# 7.DDL分区表

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

#不需要实现创建表，可以同时插入生成多个结果子表
from  TABLE_NAME1
insert override table sub_table1
select COLUMN1,COLUMN2
insert override table sub_table2
select COLUMN3
#需要实现创建结果子表
INSERT OVERWRITE TABLE psn9 SELECT id,name FROM psn
```

## 8.2通过查询结果存入本地文件/hdfs目录

很少用

**注意**：路径千万不要填写根目录，会把所有的数据文件都覆盖

```
--将查询的结果导入到本地文件系统中
insert override local directory '/root/result' select * from TABLE_NAME

--将查询到的结果导入到hdfs文件系统中
insert overwrite directory '/result' select * from psn;

```

## 8.3 数据更新和删除

**一般不用**

修改，删除动作必须保证事务开启，hdfs 文件只能appe

nd，不能修改，但是hive经过很多配置开启以后才能支持有限的修改，很多限制条件。

原理：hive把**整个文件块**复制下来，修改以后重新放回hdfs。

**因此，在使用hive的过程中，我们一般不会产生删除和更新的操作**

## 8.4 插入数据

同前面 3，5 两点。

# 9.hive运算符/函数

## 9.1关系运算符

=,<>,<,>,>=,IS NULL,IS NOT NULL 都基本等同于mysql，其他有不同如：

### like:类似mysql用法

实现左匹配，右匹配,单字符/多字符。

like "%xxx", like "xxx%", like "xx_"，以及 not like 

```

hive> select * from psn
    > where name like "小明1%";
OK
1	小明1	["王者","吃鸡"]	{"思明":"软二","集美":"软三"}
10	小明10	["war3","吃鸡"]	{"思明":"软二"}
Time taken: 0.385 seconds, Fetched: 2 row(s)


hive> select * from psn
    > where name like "%10";
OK
10	小明10	["war3","吃鸡"]	{"思明":"软二"}
20	大明10	["war3","吃鸡"]	{"思明":"软二"}

hive> select * from psn
    > where name like "小明_";
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

```

### rlike: java正则匹配,

```
如下用正则实现左，右like
hive> select 'football' rlike '^foot';
hive> select 'football' rlike 'ball$';
正则匹配数字
hive> select '2314' rlike '\\d+';

```

## 9.2算术运算符

+,-,*,/,% 以及位运算 & | ^ ~ 

## 9.3逻辑运算符

AND等意符号 && , OR 等意符号|,  

NOT：NOT A 当A为NULL 或者FALSE时才返回true，可写为!A。

## 9.4复杂类型函数

因为hive的数据类型包含复杂类型 比如 map,struct,array,所以也有对应类型的内置出函数，如:

array[n] ,Map[key],struct.x

## 9.5函数

### 数学函数

### 收集函数

主要针对map，array类型的数据 如 size()

### 类型转换函数

cast()

### 日期函数

### 条件函数

### 字符串函数

### 聚合函数

count(),sum,avg,min,max，方差 ，偏差，协方差，

### 内置表生成函数（UDTF）

  explode(array<TYPE> a)

  json_tuple

# 10.自定义函数

自定义函数包括三种UDF、UDAF、UDTF

​		UDF(User-Defined-Function) ：一进一出

​		UDAF(User- Defined Aggregation Funcation) ：聚集函数，多进一出。Count/max/min

​		UDTF(User-Defined Table-Generating Functions) :一进多出，如explore()

## 10.1 UDF 开发

1、UDF函数可以直接应用于select语句，对查询结构做格式化处理后，再输出内容。

2、编写UDF函数的时候需要注意一下几点：

a）自定义UDF需要继承org.apache.hadoop.hive.ql.UDF。

b）需要实现evaluate函数，evaluate函数支持重载。

3、步骤

(1)将jar包上传到虚拟机中：

​	a）把程序打包放到目标机器上去；

​	b）进入hive客户端，添加jar包：hive>add jar /run/jar/udf_test.jar;

​	c）创建临时函数：hive>CREATE TEMPORARY FUNCTION add_example AS 'hive.udf.Add';

​	d）查询HQL语句：

​    	SELECT add_example(8, 9) FROM scores;

​    	SELECT add_example(scores.math, scores.art) FROM scores;

​    	SELECT add_example(6, 7, 8, 6.8) FROM scores;

​	e）销毁临时函数：hive> DROP TEMPORARY FUNCTION add_example;

​	注意：此种方式创建的函数属于临时函数，当关闭了当前会话之后，函数会无法使用，因为jar的引用没有了，无法找到对应的java文件进行处理，因此不推荐使用。

(2)将jar包上传到hdfs集群中：

​	 a）把程序打包上传到hdfs的某个目录下

​	b）创建函数：hive>CREATE FUNCTION add_example AS 'hive.udf.Add' using jar "hdfs://mycluster/jar/udf_test.jar";

​	d）查询HQL语句：

​    	SELECT add_example(8, 9) FROM scores;

​    	SELECT add_example(scores.math, scores.art) FROM scores;

​    	SELECT add_example(6, 7, 8, 6.8) FROM scores;

​	e）销毁临时函数：hive> DROP  FUNCTION add_example;