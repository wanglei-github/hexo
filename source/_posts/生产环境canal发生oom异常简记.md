---
title: 生产环境canal发生oom异常简记
date: 2018-11-19 17:50:54
tags: [java,canal,OOM,RingBuffer]
---
## 背景
[canal](https://github.com/alibaba/canal)是alibaba开源一个基于binlog的增量订阅&消费组件。以一个生产者消费者的经典模式，将生产者canal-server伪装成一个mysql的从库，对binlog进行解读，消费者通过轮询方式，获取binlog的数据操作内容。利用canal可以实现很多有意思的场景。

我们在生产环境中，利用canal+kafka的架构，来实现一下报表等功能。
## OOM问题
一开始在使用canal并没有特别大的问题，配置也利用了默认配置（**多了解配置，可以帮助你更清楚的搞懂这个工具的原理**）。直到有次，线上库进行一次数据批量导入，canal的消费者异常邮件狂爆，即使备份（canal有个基于zk的冷备机制）起来了，不一会备份也会爆邮件。

查看日志，发现OOM了，canal-server在启动脚本增加了dump的命令（还是阿里大法厉害）。通过dump日志我们发现了问题原因。由于批量倒库，一条```insert values(xxx)```有1M的大小。
canal-server将消费的binlog会存储在内存中，并利用Ringbuffer一种数据结构进行存储，这种数据结构比较有意思，在很多场景中都有用到过，最著名的是[disruptor 中文版](http://ifeve.com/disruptor)。

canal-server在设置Ringbuffer的默认大小是16384，16384*1M约等于16G，内存根本吃不消，导致OOM的发生，之后调大了heap，并将Ringbuffer的大小调小到1024。

线上就再也没有发生过OOM。
