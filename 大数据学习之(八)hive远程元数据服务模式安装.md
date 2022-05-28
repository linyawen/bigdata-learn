



# 一、hive远程元数据服务模式安装：

## 1、选择两台虚拟机，node2作为服务端，node4作为客户端。



| host  | NN   | NN   | JNN  | SNN  | DN   | FC   | ZK   | RM   | NM   | H-cLI | h-sVR |
| ----- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ----- | ----- |
| node1 | *    |      | *    |      |      | *    |      |      |      | *     |       |
| node2 |      | *    | *    | *    | *    | *    | *    |      | *    |       | 1     |
| node3 |      |      | *    |      | *    |      | *    | *    | *    |       |       |
| node4 |      |      |      |      | *    |      | *    | *    | *    | 1     |       |

## 2、从node1上远程拷贝hive的安装包到Node3和node4

```
scp -r hive-2.3.7  node2:`pwd`
scp -r hive-2.3.7  node4:`pwd`

```

## 3、node2修改hive-site.xml配置：

```
	<property>
		<name>hive.metastore.warehouse.dir</name>
		<value>/user/hive_remote/warehouse</value>
	</property>
	<property>
		<name>javax.jdo.option.ConnectionURL</name>
		<value>jdbc:mysql://node01:3306/hive_remote?createDatabaseIfNotExist=true</value>
	</property>
	<property>
		<name>javax.jdo.option.ConnectionDriverName</name>
		<value>com.mysql.jdbc.Driver</value>
	</property>
	<property>
		<name>javax.jdo.option.ConnectionUserName</name>
		<value>root</value>
	</property>
	<property>
		<name>javax.jdo.option.ConnectionPassword</name>
		<value>123</value>
	</property>
```

## 4、node4修改hive-site.xml配置：

```
	<property>
		<name>hive.metastore.warehouse.dir</name>
		<value>/user/hive_remote/warehouse</value>
	</property>
	<property>
		<name>hive.metastore.uris</name>
		<value>thrift://node03:9083</value>
	</property>
```

## 5、在node2服务端执行元数据库的初始化操作，

```sql
schematool -dbType mysql -initSchema
```

## 6、在node2上启动hive的元数据服务，是阻塞式窗口

```
# 前提 ： 启动hdfs， 启动yarn
hive --service metastore
```

## 7、在node4上执行hive，进入到hive的cli窗口

```
hive> 
```





 

54:-00