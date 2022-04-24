



# 一、mapreduce 计算模型分析

map 和reduce的原理，把下面两张图看明白就基本理解了。

ps: 这两张图没有画出 **combiner**，combiner基本等同于reduce，只是执行的地方/时机不同。

好处：在map节点提前进行汇总统计，避免数据 汇总到 reduce的大量IO，也利用了map的计算资源。

注意：使用combiner提前计算是否对业务有影响。



![mapreduce图解01](https://s2.loli.net/2022/04/25/1Mkmt76yi4nDNKv.png)

下面这个图更详细且严谨

![mapreduce图解02](https://s2.loli.net/2022/04/25/xtPs93IJFH8nmBY.png)

# 二、jar包分发执行流程

![yarn图解01](https://s2.loli.net/2022/04/25/IDLd5itC3m9wQ81.png)

