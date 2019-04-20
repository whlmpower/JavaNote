# YARN 的相关整理

YARN是一个资源调度平台，负责为运算程序提供服务器运算资源，相当于分布式的操作系统平台，而MapReduce等运算程序相当于运行于操作系统之上的应用程序。

## YARN的概念

1. yarn并不清楚用户提交的程序的运行机制
2. yarn只提供运算资源的调度（用户程序向yarn申请资源，yarn就负责分配资源）
3. yarn中的主管角色叫ResourceManager
4. yarn与运行的用户程序完全解耦，意味着yarn上可以运行各种类型的分布式运算程序
5. spark，storm等运算框架都可以整合在yarn上运行，只要各自的框架中有符合yarn规范的资源请求机制即可
6. yarn成为一个通用的资源调度平台。

## YARN中运行运算程序的流程



![mapreduce&yarn的工作机制----吸星大法](.\image\mapreduce&yarn的工作机制----吸星大法.png)



1. yarnRunner（本质是Proxy） 客户端与YARN ResourceManager 通信，申请提交一个application
2. 返回一个application资源提交路径，以及application_id
3. 客户端提交job运行所需资源到指定目录中：job.split, job.xml, job.jar
4. 告诉RM资源提交完毕，申请运行MRAPPMaster
5. RM中存在资源调度策略（FIFO, FAIR, Capacity），将用户的请求初始化成一个Task，放入调度队列中
6. NodeManager领取上一步中Task
7. 根据Task描述，生成一个Container，下载Job相关资源到本地，启动MRAPPMaster（程序主管，掌握Job相关细节）
8. MRAPPMaster向RM请求运行MapTask的容器
9. NodeManager领取到任务，创建两个Container，默认进程名字叫YARNChild
10. MRAPPMaster发送程序启动脚本，mapTask启动成功，MRAPPMaster实时进行监控、重新启动等，mapTask处理成功后，在本地生成分区文件
11. MRAPPMaster 向RM申请3个容器，运行ReduceTask程序，进程默认也叫yarnChild
12. reduce向map端获取(主要通过NodeManager)相应分区的数据，使用mapReduce_shuffle组件
13. application运行完毕后，MRAPPMaster会向RM注销自己。