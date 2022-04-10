

# HDFS - HA高可用解决方案

接上一篇文章 "大数据学习之(二)hadoop完全分布式部署"，虽然说已经搭建了完全分布式集群，但是NameNode 却是单点的，一旦故障整个集群都不能访问，而且单节点内存压力也大，现在进行高可用HA 改造。

官方架构图如下：

![hdfs-ha_0](https://s2.loli.net/2022/04/10/b3sB4kOPUdx5CXG.png)

其实原理很简单，3点:

1. 再加一台NameNode2，跟原来的NameNode1通过JournalNodes集群同步数据(ap,最终一致性)。
2. 每个NameNode机器上各自启动一个辅助进程FailoveController，这两个FailoveController连到zookeeper 进行分布式抢锁，比如FC1 抢到了锁，就把本机的NameNode 设置为主，把另一台机器的NameNode设置为从（通过ssh进行设置，所以两台NameNode要配置免密）。
3. FailoveController 的职责其实就是 实现 主备2个NameNode的故障自动切换，只有 active的提供服务。

PS 疑问：

- journalNodes 看起来是不是跟 zookeeper很像？但是不一样，zk侧重分布式协调（不适合存太多数据），journalNode侧重分布式存储，用来同步NameNode的editlog 比较合适。
- 为什么hadoop的作者不把FailoveController的功能直接集成到NameNode里？简单的功能拆出多个进程不是增加运行时故障的场景吗，还有部署复杂度也增加了。难道 是为了避免nameNode的兼容性问题？

进度：

HA模式验证 18:27

## 角色规划

| host  | NN   | NN   | SNN  | DN   | FC   | ZK   |
| ----- | ---- | ---- | ---- | ---- | ---- | ---- |
| node1 | *    |      |      |      | 1    |      |
| node2 |      | 1    | *    | *    | 1    | 1    |
| node3 |      |      |      | *    |      | 1    |
| node4 |      |      |      | *    |      | 1    |



#  hadoop3.x HDFS - Federation

hadoop3.0以后出了这个nameNode 高可用+数据分片的方案，解决了 nameNode 压力过大，内存首先的问题。就跟mysql 表数据过大了以后分库分表一个道理。