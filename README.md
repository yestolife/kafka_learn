# kafka_learn
Quick start guide of Kafka

# Kafka快速入门

Kafka是一款Apache开源的消息队列。官方的介绍是这样的：

> Kafka® is used for building real-time data pipelines and streaming apps. It is horizontally scalable, fault-tolerant, wicked fast, and runs in production in thousands of companies.
>
> Kafka用于构建实时数据通道和流式应用。它具有水平扩展、容错、变态快的特点，跑在无数家公司的产品中。
![](https://kafka.apache.org/images/kafka_diagram.png)

简单来说Kafka是各应用系统消息传递的中介，类似于车间流水线上的传送带，解决了上下游工序之间的“通信”问题。
![](https://static001.geekbang.org/resource/image/84/ec/8476bca6176a7a11de452afca940feec.jpg)

Kafka涉及的概念比较多，生产者、消费者、Brocker、主题、分区等等，很容易把初学者绕晕，还不如操作一遍Kafka就知道这些概念说的是什么了。本文就手把手教大家进行Kafka基本操作，主要参考官网[Quickstart](https://kafka.apache.org/quickstart)。强烈建议在Linux下使用Kafka，因为Windows使用效果不佳，而且企业生产环境都是在Linux下，遇到问题也好找到解决办法。我们下面就正式开始吧。

## 第一步， 下载Kafka

国内从清华大学的[镜像](http://mirrors.tuna.tsinghua.edu.cn/apache/kafka/2.4.0/kafka_2.12-2.4.0.tgz)下载速度很快，或者从Apache官网[下载](https://www.apache.org/dyn/closer.cgi?path=/kafka/2.4.0/kafka_2.12-2.4.0.tgz)，下载版本2.4.0。解压tar包并进入目录。

```
> tar -xzf kafka_2.12-2.4.0.tgz
> cd kafka_2.12-2.4.0
```

## 第二步，启动服务

Kafka使用了[ZooKeeper](http://zookeeper.apache.org/)进行应用程序调度，所以要先启动ZooKeeper。正好Kafka自带了一个ZooKeeper，用如下命令启动：

```
> bin/zookeeper-server-start.sh config/zookeeper.properties
```

**注意**执行命令中可能会报无权限的错误，在命令前加`sudo`，目录前加`./`。执行`.sh`脚本前用`ll`命令查看一下该脚本是否有执行的权限。如果没有，使用chmod a+x XXX.sh命令进行提权。

接下来启动Kafka，使用如下命令：

```
> bin/kafka-server-start.sh config/server.properties
```

没有报错的话就是正常启动了。

## 第三步，创建主题

首先创建一个名为`test`的单分区、单副本的主题。

```
bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic test
```

这里涉及的概念有点多了，**主题（Topic）**是Kafka中消息队列发布和订阅的对象，主题中包含**消息**。消息是具体传输的数据。**分区**是一个有序的消息的序列。**副本**是一条消息拷贝到多个地方，提供数据冗余。
![](https://static001.geekbang.org/resource/image/06/df/06dbe05a9ed4e5bcc191bbdb985352df.png)

概念多没关系，操作起来就会清楚的。这里创建主题，就是创建一个装消息数据的容器。创建好主题后，通过如下命令可以查看主题：

```
bin/kafka-topics.sh --list --bootstrap-server localhost:9092
```

## 第四步，发送消息

Kafka提供了一个简单的命令行客户端，可以通过在界面内输入或者传文件的方式将消息发给Kafka集群。默认情况下，一行输入代表一条独立的消息。

运行生产者（producer）并输入一些消息给客户端发送至Kafka服务器。

```
> bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
This is a message
This is another message
```

生产者是指产生数据的一方。

## 第五步，启动消费者

消费者是指使用数据的一方。Kafka也提供了一个命令行客户端作为消费者，拉取消息。

```
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
This is a message
This is another message
```

## 第六步，创建多节点集群

目前我们已经跑了一个单节点broker（Kafka服务器称为broker），但没啥意思。对于Kafka，单节点只是一个节点的集群。我们来扩展集群节点到3个节点（虽然还是在本地一台机器上）。

首先将配置文件复制两份，

```
> cp config/server.properties config/server-1.properties
> cp config/server.properties config/server-2.properties
```

编辑这两个配置文件，

```
config/server-1.properties:
    broker.id=1
    listeners=PLAINTEXT://:9093
    log.dirs=/tmp/kafka-logs-1
 
config/server-2.properties:
    broker.id=2
    listeners=PLAINTEXT://:9094
    log.dirs=/tmp/kafka-logs-2
```

**注意**配置文件中log.dirs的配置与broker.id配置不在一起，虽然可以写在一起，但容易忘记下面还有log.dirs的配置没修改。

broker.id是每个集群中的节点唯一的标示。为了在同一台机器上跑三个节点，我们修改了端口和日志目录。目前已经跑起来了ZooKeeper和一个broker节点，再加两个节点：

```
> bin/kafka-server-start.sh config/server-1.properties 
...
> bin/kafka-server-start.sh config/server-2.properties 
...
```

创建一个新的主题，副本参数设置为3：

```
> bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 3 --partitions 1 --topic my-replicated-topic
```

现在我们已经有了一个Kafka集群，看看里面的主题描述：

```
> bin/kafka-topics.sh --describe --bootstrap-server localhost:9092 --topic my-replicated-topic
Topic:my-replicated-topic   PartitionCount:1    ReplicationFactor:3 Configs:
    Topic: my-replicated-topic  Partition: 0    Leader: 1   Replicas: 1,2,0 Isr: 1,2,0
```

解释一下上面的输出信息，第一行是分区（partition）的汇总描述，有一个分区，三个副本。第二行是这个分区0中具体信息，领导副本（Leader）是1号节点，副本（ Replicas）存在于1,2,0节点中，存活的节点（保持同步Isr，"in-sync" ）是1,2,0节点。

这里面又涉及到一些概念，领导副本（Leader）是对外提供消息读写服务的副本，与之对应的是跟随副本（Follower）是同步领导副本中的消息，起到备份作用，不对外提供服务的。领导副本是随机在集群中分配的节点，每次执行不一样。

向`my-replicated-topic`主题中发送一些消息：

```
> bin/kafka-console-producer.sh --broker-list localhost:9092 --topic my-replicated-topic
...
my test message 1
my test message 2
^C
```

消费这些消息：

```
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --from-beginning --topic my-replicated-topic
...
my test message 1
my test message 2
^C
```

好玩的来了，我们测试一下Kafka的容错性。Broker 1是领导副本，kill它的进程看看：

```
> ps aux | grep server-1.properties
7564 ttys002    0:15.91 /System/Library/Frameworks/JavaVM.framework/Versions/1.8/Home/bin/java...
> kill -9 7564
```

其中7564是线程遍号，每次执行都不一样。再看看主题描述：

```
> bin/kafka-topics.sh --describe --bootstrap-server localhost:9092 --topic my-replicated-topic
Topic:my-replicated-topic   PartitionCount:1    ReplicationFactor:3 Configs:
    Topic: my-replicated-topic  Partition: 0    Leader: 2   Replicas: 1,2,0 Isr: 2,0
```

领导副本已经切换为2号节点，1号节点已经不在lsr中了。消息依然可以从新的领导副本中读取，

```
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --from-beginning --topic my-replicated-topic
...
my test message 1
my test message 2
^C
```

## 总结

这个Kafka集群的搭建和测试完成了，其实很简单，是不是。别被概念绕晕了，操作一波后对kafka中的概念会有更直观的认识。深入了解Kafka和消息队列可以参考极客时间里的课程《[kafka快速入门与实战](https://time.geekbang.org/column/article/186588)》《[消息队列必知必会](https://time.geekbang.org/column/article/189275)》。







