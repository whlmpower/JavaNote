# Storm核心组件

![Storm组件图](.\image\Storm组件图.png)

1. Nimbus：负责资源分配和任务调度
2. Supervisor：负责接收nimbus分配的任务，启动和停止属于自己管理的worker进程。通过配置文件设置当前supervisor上启动多少个worker。
3. Worker：运行具体处理组件逻辑的进程。Worker运行的任务类型只有两种，一种是Spout任务，一种是Bolt任务。（其实就是一个JVM）
4. Task：worker中每一个spout/bolt线程称为一个task，在storm0.8之后，task不再与物理线程对应，不同spout/bolt的task可能会共享一个物理线程，该线程称为executor

# Storm与Hadoop区别

Storm用于实时计算，Hadoop用于离线计算。

Storm处理的数据保存在内存中，源源不断；Hadoop处理的数据保存在文件系统中，一批一批。

Storm的数据通过网络传输进来；Hadoop的数据保存在磁盘中。

Storm与Hadoop的编程模型相似

**Hadoop：**

Job：任务名称

JobTracker：项目经理

TaskTracker：开发组长、产品经理

Child：负责开发的人员

Mapper/Reducer：开发人员中的两种角色，一种是服务器开发、一种是客户端开发

**Storm：**

Topology:任务名称

Nimbus:项目经理

Supervisor:开组长、产品经理

Worker:开人员

Spout/Bolt：开人员中的两种角色，一种是服务器开发、一种是客户端开发

# Storm 编程模型

![Storm编程模型](.\image\Storm编程模型.png)

1. Topology：Storm中运行的一个实时应用程序的名称
2. DataSource：外部数据源
3. Spout：接收外部数据源的组件，将外部数据源转化为Storm内部的数据，以Tuple为基本的传输单元下方给Bolt
4. Blot：接收Spout发送的数据，或者上游Blot的发送数据。根据业务逻辑进行处理。发送给下一个Bolt或者是存储在某种介质上，介质可以是Redis或者Mysql。
5. Tuple：Storm内部中数据传输的基本单元，里面封装了一个List对象，用来保存数据。
6. StreamGrouping：数据分组策略。 7种：shuffleGrouping（Random函数），Non Grouping（Random函数）， FieldGrouping（Hash取模），Local or ShuffleGrouping本地或随机，优先本地。

**并发度：用户指定的一个任务，可以被多个线程执行，并发度的数量等于线程数量。一个任务的多个线程，会被运行在多个Worker上，有一种类似于平均算法的负载均衡策略，尽可能减少网络IO，和Hadoop中MapReduce中本地计算的道理一样**

**worker与topology**

一个worker只属于一个topology，每个worker中运行的task只能属于这个topology。

反之，一个topology中包含多个worker，其实这个topology运行在多个worker上

一个topology要求的worker数量如果不被满足，集群在任务分配时，根据现有的worker先运行topology。如果当前集群中worker数量为0，那么最新提交的topology将只会被标识为active，不会运行，只有当集群中有了充足的资源后，才会运行。

# Storm 程序执行流程

![Storm运行过程](.\image\Storm运行过程.png)

1. 客户端运行Storm jar xxx.jar 驱动类
2. nimbus进行任务分配，获取空闲的worker，分配任务
3. topology任务信息，总的task数量（相加），每个Worker获得任务数（相除），task分配以一种轮询的方式进行
4. supervisor 启动work（根据端口号）
5. DataSource发送给多个Spout，Spout发送给集群中的Blot（紫色线条）
6. Blot1进行单词切分
7. 单词根据Fields Grouping，一种hash方式将对应的单词emit发射给指定的blot2