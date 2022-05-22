# Hive远程数据库模式安装

### 安装hive的步骤：

##### 0、角色规划

node1: hive。

node3:mysql

##### 	1、解压安装

下载hive 包，解压到跟hadoop同层级目录

/opt/bigdata/hive-2.3.7

##### 	2、修改环境变量

```
		vi /etc/profile
		export HIVE_HOME=/opt/bigdata/hive-2.3.7
		将bin目录添加到PATH路径中
		source /etc/profile
```

##### 	3、修改配置文件，进入到/opt/bigdata/hive-2.3.7/conf

```java
	//修改文件名称，必须修改，文件名称必须是hive-site.xml
		mv hive-default.xml.template hive-site.xml
	//增加配置：
			进入到文件之后，将文件原有的配置删除，但是保留最后一行，
			从<configuration></configuration>，将光标移动到<configuration>这一行，
			在vi的末行模式中输入以下命令
			:.,$-1d
	//增加如下配置信息：
			<property>
				<name>hive.metastore.warehouse.dir</name>
				<value>/user/hive/warehouse</value>
			</property>
			<property>
				<name>javax.jdo.option.ConnectionURL</name>
				<value>jdbc:mysql://node3:3306/hive?createDatabaseIfNotExist=true</value>
			</property>
			<property>
				<name>javax.jdo.option.ConnectionDriverName</name>
				<value>com.mysql.jdbc.Driver</value>
			</property>
  //修改为自己的数据库账号密码
			<property>
				<name>javax.jdo.option.ConnectionUserName</name>
				<value>root</value>
			</property>
			<property>
				<name>javax.jdo.option.ConnectionPassword</name>
				<value>123</value>
			</property>
```

##### 	4、添加MySQL的驱动包拷贝到lib目录

自行下载mysql-connector-java-5.1.32-bin.jar 驱动包，放到hive-2.3.7/lib/

##### 	5、执行初始化元数据数据库的步骤

```
#node1
schematool -dbType mysql -initSchema
```

##### 	6、执行hive启动对应的服务

前提：启动hdfs，启动yarn。

执行hive 进入 hivecli

```
[root@node1 conf]# hive
which: no hbase in (/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/usr/java/default/bin:/opt/bigdata/hadoop-2.6.5/bin:/opt/bigdata/hadoop-2.6.5/sbin:/root/bin:/usr/java/default/bin:/opt/bigdata/hadoop-2.6.5/bin:/opt/bigdata/hadoop-2.6.5/sbin:/opt/bigdata/hive-2.3.7/bin)
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/opt/bigdata/hive-2.3.7/lib/log4j-slf4j-impl-2.6.2.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/opt/bigdata/hadoop-2.6.5/share/hadoop/common/lib/slf4j-log4j12-1.7.5.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.apache.logging.slf4j.Log4jLoggerFactory]

Logging initialized using configuration in jar:file:/opt/bigdata/hive-2.3.7/lib/hive-common-2.3.7.jar!/hive-log4j2.properties Async: true
Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
hive> show tables

```

看到 hive-2版本提示我们最好切到 spark引擎，hadoop已经被废弃。

##### 	7、执行相应的hive SQL的基本操作 

注意：此时hive 读取了我们环境变量的 hadoop_home, 进而读取了hdfs相关的配置，然后进行相关的hdfs读写操作。

