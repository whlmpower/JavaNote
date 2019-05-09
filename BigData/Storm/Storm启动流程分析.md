# Storm启动流程分析

1. 客户端运行storm nimbus时，会调用storm的python脚本，该脚本中为每一个命令编写一个方法，每个方法都可以生成一条响应的java命令。命令的格式如下：java -server xxx.ClassName -args   
   nimbus --> Running：java -- server backtype.storm.deamon.nimbus
   supervisor --> Running：java --server backtype.strom.daemon.nimbus

2. nimbus 启动之后，接收客户端提交任务
   命令格式：  storm jar xxx.jar xxx驱动类 参数
   Running：java -client -Dstorm.jar = storm-starter-topologies-0.9.6.jar  storm.starter.wordCountTopology wordcount-28
   该命令会执行storm-starter-topologies-0.9.6.jar中的main方法，main方法中会执行以下代码：
   StormSubmitter.submitTopology("mywordcount",config,topologyBuilder.createTopology());
   topologyBuilder.createTopology() 会将程序员编写的spout和bolt对象进行序列化。
   会将用户的jar上传到nimbus物理节点的 /export/data/storm/workdir/nimbus/inbox 目录下，并且改名，改名的规则是添加了UUID，在nimbus物理节点的 /export/data/storm/workdir/nimbus/stormdist目录下有正在运行的topology的jar包和配置文件，序列化文件

3. nimbus接受到任务之后，会将任务进行分配，会产生一个assignment对象，该对象会保存在zk中，目录是/storm/assignments，该目录只保存正在运行的topology任务。

4. supervisor通过watch机制，感知到nimbus在zk上的任务分配信息，从zk上拉取任务信息，分辨出属于自己的任务。workers=ResourceWorkerSlot[hostname = 192.168.1.106,memsize = 0,cpu=0,tasks=[1,2,3,4,5,6,7,8],jvm=<null>，nodeId=xxx，，port= 6090]，worker运行在哪台机器上，启动哪个端口号。

5. supervisor 根据自己的任务信息，启动自己的worker，并分配一个端口

   /export/data/storm/workdir/**supervisor/stormdist**/workcount1-3-122322/stormjar.jar backtype.storm.daemon.worker wordcount1-3-122322（运行的topology）   xxxUUID1（运行的supervisor物理机上） 6701（运行的端口） xxxUUID2（运行worker的标识）。在supervisor目录下的stormdist目录下的jar。

6. worker启动之后，连接zk，拉取任务ResourceWorkerSlot[hostname = 192.168.1.106,memsize = 0,cpu=0,tasks=[1,2,3,4,5,6,7,8],jvm=<null>，nodeId=xxx，port= 6090]

   假设获得的任务信息：

   1-->spot--type:spout

   2-->bolt--type:blot

   3-->acker--type:blot

   worker 通过反序列化，得到程序员自己定义的spout和bolt对象

7. worker根据任务的类型，分别执行spout任务或者bolt任务
   spout的生命周期：open、nextTuple、outPutField
   bolt的生命周期：prepare、execute(tuple)、outPutField

![Storm任务提交过程](.\image\Storm任务提交过程.png)

# Storm 组件本地目录树

![Storm组件本地目录树](.\image\Storm组件本地目录树.png)

# Strom zookeeper上的目录树

![Storm zookeeper 目录树](.\image\Storm zookeeper 目录树.png)

# Storm 通信机制

Worker间通信经常需要通过网络跨节点进行，Storm使用netty进行通信。

Worker进程内部通信：不同worker的thread通信使用Disruptor来完成，不同topology间的通信，Storm不负责，可以使用kafka实现。

## Worker进程间通信



![Worker间通信](.\image\Worker间通信.png)

对于Worker进程来说，为了管理流入和传出的消息，每个worker进程有一个独立的接收线程（对配置的TCP端口 supervisor.slots.ports进行监听）；负责将外部发送过来的消息移动到对应的executor线程的incoming-queue

对于Worker接收线程，每个Worker存在一个独立的发送线程，负责从Worker的transfer-queue中读取消息，并通过网络发送给其他worker，transfer-queue的大小由参数topology.transfer.buffer.size来进行设置。transfer-queue的每个元素实际代表一个Tuple的集合。

Worker接收线程将收到的消息通过task编号传递给对应的executor（一个或者多个）的incoming-queues；每个executor有单独的线程分别来处理spout/bolt的业务逻辑，业务逻辑输出的中间数据会存放在outgoing-queue中，当executor的outgoing-queue中的Tuple达到一定的阈值，executor的发送线程将批量获取outgoing-queue中的Tuple，并发送到transfer-queue中。

每个worker进程控制一个或者多个executor线程，用户可以在代码中进行配置，其实就是我们在代码中设置的并发度个数。

## Worker进程间通信分析

![worker通信2](.\image\worker通信2.png)

1. Worker接收线程通过网络接收数据，并根据Tuple中包含的taskId，匹配到对应的executor；然后根据executor找到对应的incoming-queue，将数据存放到incoming-queue队列中
2. 业务逻辑执行线程消费incoming-queue的数据，通过调用bolt的execute()方法，将Tuple作为参数传递给用户自定义的方法
3. 业务逻辑执行完毕后，将计算的中间数据发送给outg-queue队列，当outgoing-queue中的Tuple达到一定阈值，executor的发送线程将批量获取outgoing-queue中Tuple，并发送到Worker的transfer-queue中。
4. Worker发送线程消费transfe-queue中的数据，计算Tuple的目的地，连接不同的node+port将数据通过网络的方式传输给另一个Worker
5. 另一个Worker重复执行上述过程

![Worker通信3](.\image\Worker通信3.png)

