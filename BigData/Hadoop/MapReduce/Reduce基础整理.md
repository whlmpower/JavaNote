# MapReduce原理

MapReduce分布式程序通用框架，一个完整的MapReduce程序在分布式运行时有三类实例进程：

1. MapAppMaster（MapReduce application master）：负责整个程序的过程调度及状态协调
2. MapTask：负责map阶段的整个数据处理流程
3. ReduceTask：负责Reduce阶段整个数据处理流程

## MR程序运行流程

程序执行流程整理过程

![wordcount运行过程的解析](.\image\wordcount运行过程的解析.png)

1. 客户端获取待处理数据的信息，根据参数配置，形成一个任务分配的规划，叫job.split

2. submit 将job.split wc.jar  job.xml 提交给远端的yarn
3. YARN根据提交信息，启动MrAppMaster 起来,并将信息提交给它
4. YARN（ResourceManager）根据每台机器上面的NodeManager决定哪个为MrAppMaster
5. MrAPPMaster启动三个mapTask进程，并决定要处理分片大小
6. 尽量在同一台机器上，打开文件流，进行读取，实际读取使用InputFormat组件进行读取
7. InputFormat按照文件读取HDFS文件，每次读取一行，（k为行号，value为行的数据）读取到交给了wordcountMapper
8. context.write 交给outputController，产生结果文件（分区）
9. reduceTask等待MapTask处理完成后，从结果文件中取数据，一般情况下每个reduceTask选取一个map产生的分区
10. 拿到一组单词，调用wordCountReducer
11. 调用outPutFormat输出结果

**讲义流程**



![MR运行流程图](.\image\MR运行流程图.png)

### 流程解析

1. 一个mr程序启动的时候，最先启动的是MRAPPMaster，MRAPPMaster启动后根据本次job的描述信息，计算需要的mapTask实例数量，然后向集群申请机器启动对应数量的mapTask进程
2. MapTask进程启动之后，根据给定的数据切片范围进行数据处理，主体流程为：

(1) 利用客户指定的InputFormat来获取RecordReader读取数据，形成KV对

(2)将输入KV对传递给客户定义的map()方法，做逻辑运算，并将map()方法输出的KV对收集到缓存

(3)将缓存中的KV对按照K分区排序后不断写入到磁盘文件中

3.  MRAPPMaster监控到所有mapTask进程完成后，会根据客户指定的参数启动相应数量的reduceTask进程，并告知reduceTask进程，并告知reduceTask进程要处理的数据范围（数据分区）
4. ReduceTask进程启动后，根据MRAPPMaster告知的待处理数据所在位置，从若干台mapTask运行所在机器上获取到若干个mapTask输出结果文件，并在本地进行重新归并并排序，然后按照相同key的KV为一个组，调用客户端定义的reduce()方法进行逻辑运算，并收集运算输出的结果KV，然后调用客户指定的outputFormat将结果输出到外部存储

## MapTask并行度决定机制

一个job的map阶段并行度是由客户端在提交job时决定

客户端对map阶段并行度的规划逻辑为：将待处理数据进行逻辑切片（按照一个特定切片大小，将待处理数据划分为逻辑上多个split），每个切片分配一个mapTask并行实例处理。

切片的逻辑及切片规划描述文件，由FileInputFormat实现类的getSplit()方法完成，实现过程如下：

![切片生成逻辑](E:\MyNote\ToGitHub\BigData\Hadoop\MapReduce\image\切片生成逻辑.png)



1. 数据所在目录
2. 遍历目录下每一个文件
3. 获取文件大小，fs.sizeOf()
4. 计算切片大小computeSplitSize(Math.max(minSize, Math.max(MaxSize, blockSize))) 
5. 形成切片
6. 将切片信息写入到切片规划文件中

### FileInputFormat切片机制

#### 切片定义在InputFormat类中的getSplit()方法

#### FileInputFormat中默认的切片机制：

1. 简单地按照文件的内容长度进行切片
2. 切片大小，默认等于block大小
3. 切片时不考虑数据集整体，而是逐个针对每一个文件单独切片

#### FileInputFormat中切片的大小参数设置

在FileInputFormat中，计算切片大小的逻辑：Math.max(minSize, Math.min(maxSize, blockSize));  切片主要由这几个值来运算决定

| minsize：默认值：1              配置参数： mapreduce.input.fileinputformat.split.minsize |
| ------------------------------------------------------------ |
| maxsize：默认值：Long.MAXValue           配置参数：mapreduce.input.fileinputformat.split.maxsize |
| blockSize                                                    |

默认情况下，切片大小=blockSize

maxSize（切片最大值）：设置比blockSize小，切片会变小

minSize（切片最大值）：设置比blockSize大，切片会变大

## ReduceTask并行度的决定

手动设置

```Java
job.setNumReduceTasks(4)；
```

reduceTask数量并不可以任意设置，需要考虑业务需求，有些情况下，需要计算全局汇总结果，，就只能有1个reduceTask

## MapReduce程序运行模式

### 本地运行模式

1. MapReduce程序时被提交给LocalJobRunner在本地以单进程的形式运行
2. 处理的数据及输出结果可以在本地的文件系统，也可以在hdfs上
3. 程序编写中不要带集群的配置文件，本质是你的mr程序的conf中是否有MapReduce.framework.name=local 以及 yarn.resourcemanager.hostName参数

### 集群运行模式

1. 将MapReduce程序提交给Yarn集群resourceManager，分发到很多节点上并发执行
2. 处理的数据和输出的结果应该位于hdfs文件系统

```shell
 $ hadoop jar wordcount.jar cn.itcast.bigdata.mrsimple.WordCountDriver inputpath outputpath
```

mapreduce 在集群上运行的大体流程

## 客户端提交Job的流程

![客户端提交mr程序job的流程](E:\MyNote\ToGitHub\BigData\Hadoop\MapReduce\image\客户端提交mr程序job的流程.png)

1.jobWaitForCompletion() 调用job.submit( ) 调用JobSubmiter,

2.JobSubmiter 成员Cluster，Cluster也有一个成员Proxy

3.最后提交到yarn上面，这时创建的Proxy为YarnRunner；提交到本地（MR 程序运行模拟器）,创建的Proxy为LocalJobRunner

4.通过Runner拿到提交资源的路径，StagingDir--> File:/.../.staging ;  hdfs:// ..../.staging  

5.拿到JobID，连接StagingDir，拼接成Job资源提交路径  hdfs:// ..../.staging /jobid

6.将Job资源提交到对应的目录：切片的规划（调用FileInputFormat.getSplit()，得到List<FileSplit>, 序列化成文件）

7.将Job相关参数写Job.xml文件，将文件上传到对应Job提交路径

8.将获取程序Job的Jar 包(Main() 方法中setJobClass() 方法)进行提交

## MapReduce中的Combiner

1. Combiner 是MR程序中Mapper和Reducer之外的一种组件

2. Combiner组件的父类是Reducer

3. Combiner和Reducer的区别在于运行的位置

   Combiner是在每一个MapTask所在的节点运行

   Reducer是接收全局所有Mapper的输出结果

4. Combiner的意义在于对每一个mapTask的输出进行局部汇总，以减少网络传输量（自定义Combiner继承Reducer，重写Reduce方法；在job中设置，job.setCombinerClass）

5. Combiner能够应用的前提是不能影响最终的业务逻辑，而且Combiner的输出KV应该跟Reducer的输入KV类型要对应起来。



