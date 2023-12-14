# 机器IP



| 机器  | ip1           | ip2            |
| ----- | ------------- | -------------- |
| node1 | 192.168.130.3 | 192.168.56.103 |
| node2 | 192.168.130.4 | 192.168.56.104 |
| node3 | 192.168.130.6 | 192.168.56.102 |
| node4 | 192.168.130.7 | 192.168.56.101 |
| node5 |               |                |
| node6 |               |                |



# 一、启动HDFS 

## 启动ZK、JN、HDFS

```
#node2,3,4
zkServer.sh start

#node1
start-dfs.sh

#如果jps看不到JN进程，则 在node1，node2,node3 依次执行 
hadoop-daemon.sh start journalnode
```

验证：访问**Hdfs web UI**控制台,主/备 node1/node2:

```
#node1
http://192.168.56.103:50070/dfshealth.html#tab-overview
#node2
http://192.168.56.104:50070/dfshealth.html#tab-overview
```

# 二、Yarn

启动NodeManager
```
#node1执行
start-yarn.sh
```
启动resourceManager

```
#node3：
yarn-daemon.sh start resourcemanager
#node4:
yarn-daemon.sh start resourcemanager
```

## 验证： Yarn WebUI

- node3  http://192.168.56.102:8088 
- node4  http://192.168.56.101:8088 

注意：Memory Total，VCores Total显示有问题，需要改配置。

# 三、hive

## 0.检查node3 mysql是否已经启动

## 1. --deprecated远程数据库模式

```shell
#检查node3 mysql是否已经启动
#node1 直接输入hive
hive
#即可使用hive sql
hive> show tables
```

## 2. 建议：远程元数据服务模式

```shell
#检查node3 mysql是否已经启动
#node2
hive --service metastore 

```

## 3.使用hive

### (1)cli

```shell
#node4 cli使用live
#其实就是 hive --service cli
hive
```

### (2)DBC方式：hiveserver2

```shell
#node4 供jdbc连接
hive --service hiveserver2
```

另找一台发起jdbc连接

```shell
#node4 新开个窗口之（或者其他）执行 beeline 连到hiveserver2
beeline
#输入jdbc
beeline> !connect jdbc:hive2://node4:10000
#用户root，密码随便

```

#### FAQ

1. User: root is not allowed to impersonate root (state=08S01,code=0)

   解决：

   ```shell
   #https://cwiki.apache.org/confluence/display/Hive/Setting+Up+HiveServer2
   #conf/hive-site.xml
   <property>
   <name>hive.server2.enable.doAs </name>
   <value>false</value>
   </property>
   
   然后使用命令bin/hiveserver2重新启动hiveserver2服务,然后使用beeline连接hiveser2，然后对表进行增删改查都不会报错了，同时使用beeline时都可以不用指定用户和密码了(bin/beeline -u jdbc:hive2://hdp01:10000)。
   ```

   

### (4)eb GUI接口（hwi、hue等）

# 四、HBase

```
#node1
start-hbase.sh
```

