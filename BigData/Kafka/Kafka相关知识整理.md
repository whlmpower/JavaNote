# Kafka相关知识整理

## Kafka是什么

类JMS消息队列，结合JMS中的两种模式，可以有多个消费者主动拉取数据，而在JMS中只有点对点模式才有消费者主动拉取数据。

Kafka是一个生产-消费模型。

Producer：生产者，只负责数据生产，生产者的代码可以集成到任务系统中。

Broker：当前服务器上的Kafka进程，只负责数据的存储，不关心数据的生产者和数据消费者。

Topic：目标发送的目的地（消息主题），这是一个逻辑上的概念，对应落实在磁盘上是一个partition的目录。partition的目录中有多个segment组合（index, log）。一个Topic对应多个partition[0,1,2,3]，一个partition对应多个segment组合，一个segment的默认大小为1G。每个partition可以设置多个副本（replication-factor  1），会从所有副本中选取一个leader出来，所有的读写操作都是通过leader进行的。特别强调，和mysql中主从有区别，mysql做主从主要是为了读写分离，在kafka中的读写都是leader中进行的。

ConsumerGroup：数据消费者组，一个topic可以有多个ConsumerGroup。topic的消息会复制（不是真的复制，只是逻辑上的复制）到所有的ConsumerGroup，但是每个partition只会把消息发送给该CG中的一个consumer。可以把多个consumer线程分为一个组，组内的所有成员共同消费通一个topic的数据，组员之间不能够重复消费。如果要实现广播，只要每一个consumer有一个独立的CG就可以了，如果要实现单播，只要所有的consumer在同一个CG。

## Kafka生产数据时的分组策略

默认是defaultPartition  Utils.abs(key.hashCode) % numPartitions

上文中的key是producer在发送数据的时候传递的，producer.send(KeyedMessage(topic,myPartitionKey,messageContent))

## Kafka如何保证数据的完全生产

ack机制：broker表示标识发送来的数据已确认接收无误，表示数据已经存储到了磁盘

0 不等待broker返回确认消息

1 等待topic中某个partition leader保存成功的状态反馈

-1 等待topic中某个partition所有读本都保存成功的状态反馈



## broker如何保存数据

在理论环境下，broker按照顺序读写机制，可以每秒保存600M的数据。主要通过pagecache机制，尽可能的利用当前物理机器上的空闲内存来做缓存。当前topic所属的broker，必定有一个该topic的partition，partition是一个磁盘目录。partition的目录中有多个segment组合（index,log）。

## partition如何分布在不同的broker上

一种类似hash的分布算法

```Java
int i = 0 ;
list<kafka01, kafka02, kafka03>;
for(int i = 0; i < 5; i++){
    brIndex = i % broker;
    hostName = list.get(brIndex);
   
}
```

## ConsumerGroup 中的Consumer和partition之间的负载均衡

最好的方式是Consumer与Partition之间是一一对应的关系；如果Consumer的数量过多，必然会有空闲的Consumer。

算法：

		假如topic1,具有如下partitions: P0,P1,P2,P3
		加入group中,有如下consumer: C1,C2
		首先根据partition索引号对partitions排序: P0,P1,P2,P3
		根据consumer.id排序: C0,C1
		计算倍数: M = [P0,P1,P2,P3].size / [C0,C1].size,本例值M=2(向上取整)
		然后依次分配partitions: C0 = [P0,P1],C1=[P2,P3],即Ci = [P(i * M),P((i + 1) * M -1)]
## 如何保证kafka消费者消费数据是全局有序的

这是一个伪命题。

如果要保证有序，必须保证生产者有序，存储有序，消费有序。

由于生产可以做集群，存储可以做分片，消费可以设置为一个ConsumerGroup，要保证全局有序，就需要保证每个环节都有序。只有一种保证有序的可能，就是只存在一个生产者，一个partition，一个消费者，但这种场景和大数据应用场景相悖。