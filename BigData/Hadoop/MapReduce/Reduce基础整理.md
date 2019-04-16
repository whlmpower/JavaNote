# MapReduce原理

MapReduce分布式程序通用框架，一个完整的MapReduce程序在分布式运行时有三类实例进程：

1. MapAppMaster（MapReduce application master）：负责整个程序的过程调度及状态协调
2. MapTask：负责map阶段的整个数据处理流程
3. ReduceTask：负责Reduce阶段整个数据处理流程

## MR程序运行流程



![wordcount运行过程的解析](.\image\wordcount运行过程的解析.png)

1. 客户端获取待处理数据的信息，根据参数配置，形成一个任务分配的规划，叫job.split

2. submit 将job.split wc.jar  job.xml 提交给远端的yarn
3. YARN根据提交信息，启动MrAppMaster 起来,并将信息提交给它
4. YARN（ResourceManager）根据每台机器上面的NodeManager决定哪个为MrAppMaster
5. MrAPPMaster启动三个mapTask进程，并决定要处理分片大小
6. 尽量在同一台机器上，打开文件流，进行读取，实际读取使用InputFormat组件进行读取
7. InputFormat按照文件读取HDFS文件，每次读取一行，（k为行号，value为行的数据）读取到交给了wordcountMapper
8. context.write 交给outputController，产生结果文件（分区）
9. reduceTask等待MapTask处理完成后，从结果文件中取数据
10. 拿到一组单词，调用wordCountReducer
11. 调用outPutFormat输出结果









![MR运行流程图](.\image\MR运行流程图.png)

