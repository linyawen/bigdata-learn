# 一 、环境准备

1.虚拟机centos 7.9，网络模式nat
2.hadoop2.6.5  老的总是稳定的。
先部署一个单机伪分布式，验证配置正确后再改为真分布式。
按照这个教程，基本上能把hadoop单机伪分布式部署起来，并且试运行一个官方demo：wordCount。

## 基础设施 

### 设置ip 

vi /etc/sysconfig/network-scripts/ifcfg-enp0s3

```
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="static"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="enp0s3"
UUID="150fc384-e53f-4d69-a0e1-6939107bfa5b"
DEVICE="enp0s3"
ONBOOT="yes"
IPADDR="192.168.130.3"
NETMASK=255.255.255.0
GATEWAY=192.168.130.2
DNS1=114.114.114.114
```

service network restart

### 设置主机名

```
hostnamectl set-hostname node1
```

### 设置host

```
vi /etc/hosts
192.168.130.3 node1
```

### 关闭防火墙和 selinux

```
systemctl stop firewalld
systemctl disable firewalld

setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/sysconfig/selinux

```

### 时间同步

```
yum install -y ntp
vi /etc/ntp.conf
#增加一行
server ntp1.aliyun.com
service ntpd start
systemctl enable ntpd
```

### 安装jdk

```
yum install -y java-1.8.0-openjdk java-1.8.0-openjdk-devel
# 设置JAVA_HOME
vi /etc/profile
	export JAVA_HOME=/usr/java/default
	export PATH=$PATH:$JAVA_HOME/bin
source /etc/profile
	
mkdir -p /usr/java/
#根据自己的安装路径设置jdk软连接
ln -s /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.322.b06-1.el7_9.x86_64 /usr/java/default

```

### ssh免密

验证自己是否免密:

```
  
 ssh localhost
 #此时自动生成 /root/.ssh目录
```

生成密钥

```
ssh-keygen -t dsa -P '' -f ~/.ssh/id_dsa
```

分发公钥

（此处用单机伪分布式的方式分发公钥）

```
cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys
```

# 二、hadoop搭建配置

## 解压程序

mkdir /opt/bigdata

解压hadoop-2.6.5 到本目录下

## 设置HADOOP_HOME

```
vi /etc/profile
    export JAVA_HOME=/usr/java/default
    export HADOOP_HOME=/opt/bigdata/hadoop-2.6.5
    export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
    
source /etc/profile
```

## 目录说明

```
cd   $HADOOP_HOME
--sbin
#各种角色的启停脚本
----start--xxxx.sh
----stop--xxxx.sh
--bin
#hadoop的各种功能模块
----
```

## 配置hadoop

### 设置java_home

ssh执行脚本时读取不到系统环境变量，所以要这么设置

```
cd $HADOOP_HOME/etc/hadoop
vi hadoop-env.sh
	export JAVA_HOME=/usr/java/default
```

### 设置hdfs-namenode的ip/port

```
vi core-site.xml
<configuration>
        <property>
                <name>fs.defaultFS</name>
                <value>hdfs://node1:9000</value>
        </property>
</configuration>

```

### 设置dfs参数

文件块副本数（目前伪分布式用1，真分布式改成2、3）,namenode,datanode的数据存储路径, secondNameNode的ip/port

```
vi hdfs-site.xml
<configuration>
			<property>
				<name>dfs.replication</name>
				<value>1</value>
			</property>
			<property>
				<name>dfs.namenode.name.dir</name>
				<value>/var/bigdata/hadoop/local/dfs/name</value>
			</property>
			<property>
				<name>dfs.datanode.data.dir</name>
				<value>/var/bigdata/hadoop/local/dfs/data</value>
			</property>
			<property>
				<name>dfs.namenode.secondary.http-address</name>
				<value>node1:50090</value>
			</property>
			<property>
				<name>dfs.namenode.checkpoint.dir</name>
				<value>/var/bigdata/hadoop/local/dfs/secondary</value>
			</property>
</configuration>
 
```

### 分配dataNode 服务器

```
vi slaves
	node1
```

# 三、 dfs初始化&启动

## 格式化namenode

仅仅部署时执行一次，不可多次使用，会清除数据。（是否应该先备份这个local/dfs/name目录）

```
hdfs namenode -format 
```

显示如下信息，已经根据hdfs-site.xml中配置的路径信息，初始化好namenode的路径以及数据。

```
22/04/05 20:12:31 INFO common.Storage: Storage directory /var/bigdata/hadoop/local/dfs/name has been successfully formatted.
22/04/05 20:12:31 INFO namenode.FSImageFormatProtobuf: Saving image file /var/bigdata/hadoop/local/dfs/name/current/fsimage.ckpt_0000000000000000000 using no compression
22/04/05 20:12:31 INFO namenode.FSImageFormatProtobuf: Image file /var/bigdata/hadoop/local/dfs/name/current/fsimage.ckpt_0000000000000000000 of size 321 bytes saved in 0 seconds.
22/04/05 20:12:31 INFO namenode.NNStorageRetentionManager: Going to retain 1 images with txid >= 0
22/04/05 20:12:31 INFO util.ExitUtil: Exiting with status 0
22/04/05 20:12:31 INFO namenode.NameNode: SHUTDOWN_MSG: 
/************************************************************
SHUTDOWN_MSG: Shutting down NameNode at node1/192.168.130.3
************************************************************/

```

简单看下namenode的相关信息

```
[root@node1 current]# pwd
/var/bigdata/hadoop/local/dfs/name/current
[root@node1 current]# ll
total 16
-rw-r--r--. 1 root root 321 Apr  5 20:12 fsimage_0000000000000000000
-rw-r--r--. 1 root root  62 Apr  5 20:12 fsimage_0000000000000000000.md5
-rw-r--r--. 1 root root   2 Apr  5 20:12 seen_txid
-rw-r--r--. 1 root root 205 Apr  5 20:12 VERSION

```

可以看到VERSION里有集群id

```
less VERSION

namespaceID=1765714376
clusterID=CID-22c92132-49d5-45d3-bed4-a4d860d5578a
cTime=0
storageType=NAME_NODE
blockpoolID=BP-869714883-192.168.130.3-1649160751195
layoutVersion=-60

```

## 启动dfs

执行脚本，里面会用通过ssh免密启动所有角色

- namenode(读取配置 core-site.xml取到节点)
- datanode (读取配置slaves文件取到节点 )
-  secondNameNode(读取hdfs-site.xml 取到节点)

```
start-dfs.sh
#关闭集群 stop-dfs.sh
```

可以看到已经初始化好dataName, secondaryNameNode的目录。

```
[root@node1 current]# cd /var/bigdata/hadoop/local/dfs/
[root@node1 dfs]# ll
data  name  secondary
[root@node1 dfs]# 
```

并且看到dataNode已经跟nameNode建立了连接，同步了clusterID

```
cd /var/bigdata/hadoop/local/dfs/data/current
[root@node1 current]# cat VERSION 
#Tue Apr 05 20:59:40 CST 2022
storageID=DS-c10d6155-08f2-4818-b342-0963334047df
clusterID=CID-22c92132-49d5-45d3-bed4-a4d860d5578a
cTime=0
datanodeUuid=31ff8641-d39a-446b-9f81-a6d0a5238e41
storageType=DATA_NODE
layoutVersion=-56

```

secondNameNode 也是做了类似操作。

## Web后台

浏览器打开hdfs的后台 http://127.0.0.1:50070/

也可以看secondNameNode的后台，不过没什么功能。

# 四、dfs简单使用

命令行操作dfs

## 命令帮助

```
[root@node1 tmp]# hdfs dfs
Usage: hadoop fs [generic options]
	[-appendToFile <localsrc> ... <dst>]
	[-cat [-ignoreCrc] <src> ...]
	[-checksum <src> ...]
	[-chgrp [-R] GROUP PATH...]
```

## 新建目录

```
hdfs dfs -mkdir /bigdata
```
把虚拟机的50070端口映射到宿主机上，就可以用浏览器访问hdfs后台如下

![hdfs新建目录1](https://s2.loli.net/2022/04/07/8oTYuNi2lMRxrKH.png)

一般会在hdfs新建个home目录，

```
hdfs dfs -mkdir -p /user/root
# root表示客户端的用户名
```

## 上传文件

默认分片大小 128MB

```
hdfs dfs -put hadoop-2.6.5.tar.gz  /user/root
```

![hdfs上传文件1](https://s2.loli.net/2022/04/07/CHd2Rl7unxs6JW5.png)

![hdfs文件块信息1](https://s2.loli.net/2022/04/07/z6pldNa9kBFDsP2.png)

在dataNode节点的存储路径上可以查看上图block0对应的linux文件块在哪里

```
cd /var/bigdata/hadoop/local/dfs/data/current/BP-869714883-192.168.130.3-1649160751195/current/finalized/subdir0/subdir0

[root@node1 subdir0]# ll
total 180700
-rw-r--r--. 1 root root 134217728 Apr  5 22:12 blk_1073741825
-rw-r--r--. 1 root root   1048583 Apr  5 22:12 blk_1073741825_1001.meta
-rw-r--r--. 1 root root  49377148 Apr  5 22:12 blk_1073741826
-rw-r--r--. 1 root root    385767 Apr  5 22:12 blk_1073741826_1002.meta

```

## 自定义块大小

生成测试文本

```
cd /opt/bigdata/test
for i in `seq 100000`;do  echo "hello hadoop $i"  >>  data.txt  ;done
```

```
# 默认推送到home目录 /user/root
hdfs dfs -D dfs.blocksize=1048576  -put  data.txt
```

# 五、运行官方 Demo-wordCount

1.建立wordCount工作目录

```
hdfs dfs -mkdir -p   /data/wc/input
```

2.上传测试文件到工作目录

```
cd /opt/bigdata/test
hdfs dfs -D dfs.blocksize=1048576  -put data.txt  /data/wc/input
```

3.执行官方example的jar包

```
cd  $HADOOP_HOME
cd share/hadoop/mapreduce
hadoop jar  hadoop-mapreduce-examples-2.6.5.jar   wordcount   /data/wc/input   /data/wc/output
```

注意：

- output  必须是空的，否则会报错。
- 这种方式适合开发使用

4.查看结果

```
[root@node1 mapreduce]# hdfs dfs -ls /data/wc/output
Found 2 items
-rw-r--r--   1 root supergroup          0 2022-04-06 02:12 /data/wc/output/_SUCCESS
-rw-r--r--   1 root supergroup     788922 2022-04-06 02:12 /data/wc/output/part-r-00000

```

```
#输出统计以后的结果
hdfs dfs -cat  /data/wc/output/part-r-00000
#保存文件到本地磁盘
hdfs dfs -get  /data/wc/output/part-r-00000  ./
```

5.重新执行的话要先清理结果目录

```
hdfs dfs -rm -r /data/wc/output
```

# 六、todo

多节点分布式、yarn, hive,spark