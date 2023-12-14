# 机器IP



|       node1      | node2            |
| ----- | ------------- |
|   master | master |
| namesvr | namesvr |
| broker | broker（disk no space） |



# 一、双 master模式
## 部署

```shell
#ps: 可以先处理好node1，然后 复制到node2
#scp -r ./rocketmq/ node2:`pwd`
#1、下载mq到node1，2
cd /opt/rocketmq
wget https://archive.apache.org/dist/rocketmq/4.9.4/rocketmq-all-4.9.4-bin-release.zip
unzip rocketmq-all-4.9.4-bin-release.zip
cd rocketmq-all-4.9.4-bin-release/bin
#2、设置ROCKETMQ_HOME
vi /etc/profile
export ROCKETMQ_HOME=/opt/rocketmq/rocketmq-all-4.9.4-bin-release
source /etc/profile
#3、调低jvm内存，不然namesvr默认4G，broker 8G
cd $ROCKETMQ_HOME/bin
vi runserver.h
vi runbroker.h

#4、启动Name Server node1，2
$ nohup sh mqnamesrv &
#5， 验证Name Server 是否启动成功
$ tail -f ~/logs/rocketmqlogs/namesrv.log
The Name Server boot success...

#6、broker配置文件修改 node1，2
##6.1 2个namesvr地址（命令行 -n 不能传入2个namesvr）
##6.2 监听公网ip（便于开发调试）
### node1
#### namesrvAddr=node1:9876;node2:9876
#### brokerIP1=192.168.56.103
vi ../conf/2m-noslave/broker-a.properties
### node 2
#### namesrvAddr=node1:9876;node2:9876
#### brokerIP1=192.168.56.104
vi ../conf/2m-noslave/broker-b.properties

#7、启动broker
##6.2 node1，启动第一个Master，例如NameServer的IP为：192.168.1.1
$ nohup sh mqbroker  -c $ROCKETMQ_HOME/conf/2m-noslave/broker-a.properties &
##6.3 node2，启动第二个Master，例如NameServer的IP为：192.168.1.1
$ nohup sh mqbroker -c $ROCKETMQ_HOME/conf/2m-noslave/broker-b.properties &


```

## 启动

```shell
#一、启动Name Server node1,node2  
nohup sh mqnamesrv &
#二、启动broker
#1、node1 
nohup sh mqbroker  -c $ROCKETMQ_HOME/conf/2m-noslave/broker-a.properties &
#2、node2
nohup sh mqbroker -c $ROCKETMQ_HOME/conf/2m-noslave/broker-b.properties &
```

## WebUI

http://192.168.56.104:8080/#/

## 关闭

```shell
sh mqshutdown broker
sh mqshutdown namesrv
```



# 二、运维工具dashboad 和 mqadmin

## （一）dashboard

两种部署方式

### 1.docker部署

```shell
#node2里部署dashboard镜像
docker pull apacherocketmq/rocketmq-dashboard:latest

#注意，网络模式要用host，否在阿里云上可能访问不到本机的nameserver9876
docker run --net=host  -d --name rocketmq-dashboard -e "JAVA_OPTS=-Drocketmq.namesrv.addr=192.168.130.3:9876" -p 8080:8080 -t apacherocketmq/rocketmq-dashboard:latest
```



### 2.webUI

http://192.168.56.104:8080/#/

设置账号密码

todo

### 3.jar包部署

```
#node1部署jar包
```

## （二）mqadmin

### 1.statsAll指标监控

```
 sh mqadmin  statsAll -n 192.168.130.3:9876

```

### 2.clusterList

```

```

### 3.clusterRT

查看broker是否能接收消息，延时多少

### 4.消息查看

proceduer 发送消息后返回的对象例如：

```shell
SendResult [sendStatus=SEND_OK, msgId=7F00000138A018B4AAC2476038060007, offsetMsgId=C0A8386700002A9F0000000000295E68, messageQueue=MessageQueue [topic=TopicTest3, brokerName=broker-a, queueId=3], queueOffset=351]
```

#### a.uniqueKey

其中msgId=7F00000138A018B4AAC2476038060007 为mqadmin中使用的uniqueKey,而非queryMsgById中使用的Id,

应该使用配对的queryMsgByUniqueKey 入：

```shell
mqadmin queryMsgByUniqueKey -t TopicTest3 -n localhost:9876 -i 7F00000138A018B4AAC2476038060007
```

#### b.msgId

其中offsetMsgId=C0A8386700002A9F0000000000295E68为 mqadmin中使用的msgId，如:

```shell
mqadmin queryMsgById -n localhost:9876 -i C0A8386700002A9F0000000000295E68
```

#### c.queryMsgByKey 

```shell
mqadmin queryMsgByKey -k messageKey4_7 -t TopicTest3 -n localhost:9876
```






# 三、测试

## 收发消息可用

https://rocketmq.apache.org/zh/docs/4.x/introduction/02quickstart

```shell
$ export NAMESRV_ADDR=localhost:9876
$ sh bin/tools.sh org.apache.rocketmq.example.quickstart.Producer
 SendResult [sendStatus=SEND_OK, msgId= ...

$ sh bin/tools.sh org.apache.rocketmq.example.quickstart.Consumer
 ConsumeMessageThread_%d Receive New Messages: [MessageExt...


```

# 四、事务消息

[RocketMQ进阶 - 事务消息 - 知乎 (zhihu.com)

[[RocketMQ的事务消息机制 - 简书 (jianshu.com)](https://www.jianshu.com/p/db06e4037ac1)](https://zhuanlan.zhihu.com/p/159573084)

![](D:\学习资料\大数据学习笔记\static\rocketmq_transaction.png)

## 消息重试

![](D:\学习资料\大数据学习笔记\static\rocketmq_mq_retry.png)



# 五、死信

如果产生了死信消息，那对应的ConsumerGroup的死信Topic名称为%DLQ%ConsumerGroupName，死信队列的消息将不会再被消费。可以利用RocketMQ Admin工具或者RocketMQ Dashboard上查询到对应死信消息的信息

## （一）代码用例

### 1.只会重试，不会进死信队列

消费代码被异常打断 throw new RuntimeException("err msg");

### 2.会进死信队列

消费代码返回return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;

### 3.代码范例

```java
/*
         *  Register callback to execute on arrival of messages fetched from brokers.
         */
        consumer.registerMessageListener((MessageListenerConcurrently) (msg, context) -> {
            System.out.printf("%s Receive New Messages: %s %n", Thread.currentThread().getName(), msg);
            return ConsumeConcurrentlyStatus.RECONSUME_LATER;
            //直接抛异常不会走入死信队列
//   throw new RuntimeException("err msg");
            //必须返回指定值,才会进入私信队列
//            return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
        });
```

## （二）查询死信消费者主题

```shell
[root@node2 bin]# sh mqadmin  topicList -n 192.168.130.3:9876 |grep DLQ
%DLQ%please_rename_unique_group_name_4
#死信Topic名称为%DLQ%ConsumerGroupName


```

## （三）查询死信 数量

用mqadmin  打印出 **死信消费者主题** 下的消息列表，并过滤出MSGID、优先分析死信多的主题。

```shell
sh mqadmin  printMsg -n 192.168.130.3:9876  -t   %DLQ%please_rename_unique_group_name_4|grep MSGID: |wc -l
```

## （四）抽取死信 的关键信息

### 1.抽取死信的topic 

分组统计死信**最多的topic，优先处理**

```shell
#输出所有的topic，大概看看数量
sh mqadmin  printMsg -n 192.168.130.3:9876  -t   %DLQ%please_rename_unique_group_name_4|grep -P "RETRY_TOPIC=(.+?)\," -o
```

```shell
#按topic分组统计
sh mqadmin  printMsg -n 192.168.130.3:9876  -t   %DLQ%please_rename_unique_group_name_4|grep -P "RETRY_TOPIC=(.+?)\," -o|
awk -F '|' '{count[$1]++;} END {for(i in count) {print i count[i]}}'
#输出 219 个。
RETRY_TOPIC=TopicTest,219


```


### 2.抽取指定topic的死信

**导出**指定topic的死信body，用以重发

```shell
sh mqadmin  printMsg -n 192.168.130.3:9876  -t   %DLQ%please_rename_unique_group_name_4| grep  "RETRY_TOPIC=TopicTest"|grep -P -o "BODY:(.*$)" 
#显示
BODY: Hello RocketMQ 63
BODY: Hello RocketMQ 39
BODY: Hello RocketMQ 31
```

### PS: grep 

 -P 非贪婪正则 

 -o 只显示匹配部分。

##  (七)处理死信

https://blog.csdn.net/Decade_Faiz/article/details/131007843

思路就是把死信落库，然后根据实际情况重发。

### 1.编写消费者监听死信队列

#### ~~缺点~~

就是如果很多Topic都产生了死信消息，那么我们想要处理这些死信消息就得编写很多个监听各个死信队列的消费者。

#### 缺点解决

用一个consumer多次subscribe多个 DLQ队列，来批量管理多个死信主题。如下

```
// 动态设定topic的订阅，以及动态注册监听
	// 注意：每调用一次setConsumer方法，topic监听会累计而不是覆盖
    public static void setConsumer(String topic) {

            try {
                consumer.subscribe(topic, "*");
                consumer.registerMessageListener(new MessageListenerConcurrently() {

                    public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,
                                                                    ConsumeConcurrentlyContext context) {
                        System.out.printf("\r\n%s Receive New Messages: %s %n", Thread.currentThread().getName(), msgs);
                        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
                    }
                });
            } catch (MQClientException e) {
                e.printStackTrace();
            }


    }
```



### 2.重试到一定次数时，直接落库。

```java
consumer.registerMessageListener(new MessageListenerConcurrently() {
        @Override
        public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> list, ConsumeConcurrentlyContext consumeConcurrentlyContext) {
            MessageExt messageExt = list.get(0);
            System.out.println(new Date());
            try {
                handleDb();// 10/0
            } catch (Exception e) {
                if (messageExt.getReconsumeTimes() >= 2) {
                    //不要重试了
                    System.out.println("消息体：" + new String(messageExt.getBody()));
                    System.out.println("记录到特别的位置如mysql，发送邮件或短信通知人工处理");
                } else {
                    //重试
                    System.out.println("时间：" + new Date() + "\t消息体：" + new String(messageExt.getBody()) + "\t重试次数：" + messageExt.getReconsumeTimes());
                    return ConsumeConcurrentlyStatus.RECONSUME_LATER;
                }
            }
            return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
        }
    });
    consumer.start();

```

#### 优点

不用每个topic consumer 都写死信主题监听代码。



### 3.TODO

重新消费指定区间的消息

consumeMessage

# 六、消息堆积查看

## 1.dashboard:

主题  -> 主题列表行 -> consumer管理 ：订阅组（延迟）

消费者->  订阅组(行)->消费详情

## 2.mqadmin

consumeGroup维度进行查询，查询该consumeGroup下的所有topic的所有queue的写入进度和消费进行比对。

```shell
sh mqadmin consumerProgress -n localhost:9876 -g consumer_group_test
```

# 七、消息轨迹

默认情况下，RocketMQ 是不开启轨迹消息的，需要我们手工开启

## 1.broker开启

```
traceTopicEnable=true
```

## 2.proceduer开启

对于生产者端，要开启轨迹消息，需要在定义生产者时增加参数。定义消费者使用类 DefaultMQProducer，这个类支持开启轨迹消息的构造函数如下

```java

//ture代表开启，null表示使用默认轨迹队列RMQ_SYS_TRACE_TOPIC
DefaultMQProducer producer = new DefaultMQProducer(PRODUCER_GROUP2,true,null);
```

## 3.consumer 开启

```java
//ture代表开启，null表示使用默认轨迹队列RMQ_SYS_TRACE_TOPIC
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer(CONSUMER_GROUP2,true,null);
```

## 4.效果

![](D:\学习资料\大数据学习笔记\static\rocketmq_msg_trace.png)

![](D:\学习资料\大数据学习笔记\static\rocketmq_msg_trace_2.png)

[5 张图带你彻底理解 RocketMQ 轨迹消息-鸿蒙开发者社区-51CTO.COM](https://ost.51cto.com/posts/25194)

# FAQ

## 1.broker磁盘不足以后该broker不可用。

表现：

- 如果没有其它的可用broker，则  client端报错 service not available。
- 如果有其它brokerB可用，则
  - client段不会报错。
  - 知会发导brokerB的队列。
  - mqadmin  clusterRT 指令会发现该broker 探测失败。

解决：df -h 查看磁盘暂用率 对比 参数配置diskMaxUsedSpaceRatio 

## 2.FAQ from github

https://rocketmq.apache.org/docs/bestPractice/06FAQ/

## 3.208 DESC

[Rocketmq-console 重发消息失败： 208 DESC-CSDN博客](https://blog.csdn.net/zhangjun039009/article/details/130400691)

## 

