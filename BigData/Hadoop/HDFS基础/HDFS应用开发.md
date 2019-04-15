# HDFS应用开发

## HDFS的Java操作

在Java中操作HDFS，首先获得客户端实例

```java
Configuration conf = new Configuration()
FileSystem fs = FileSystem.get(conf)
```

从conf中的参数fs.defaultFS配置

若代码中没有指定fs.defaultFS，并且工程的classpath下也没有给定相应的配置，conf的默认值来自于Hadoop的jar包中的core-default.xml，默认值为file:///，则获取的将不是一个DistributedFileSystem，而是一个本地文件系统的客户端对象。

## HDFS客户端操作数据代码示例

```Java
public void init(){
    Configuration conf = new Configuration();
    conf.set("fs.defaultFS", "hdfs://hdp-node01:9000");
    conf.set("dfs.replication", 3);
    
    fs = FileSystem.get(conf)；//获取访问hdfs的访问客户端，根据参数，这个实例为DistributedFileSystem的实例
        
    fs = FileSystem.get(new URI("hdfs://hdp-node01:9000", conf, "hadoop"));//这种设置，conf中就不需要配置ds.defaultFS参数，而且客户端标识为Hadoop用户
    
}
```

configuration 参数设置：hdfs的URI，从而FileSystem.get()方法就知道应该去构造hdfs文件的客户端，new Configuration 的时候，会加载jar包中的hdfs-default.xml

参数优先级：1.客户端代码中设置的值；2.classpath下的用户定义配置文件；3.服务器的默认配置。

## shell 采集脚本

```shell
#! /bin/bash

#set java env
export JAVA_HOME=/home/hadoop/app/jdk1.7.0_51
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export PATH=${JAVA_HOME}/bin:$PATH

#set hadoop env
export HADOOP_HOME=/home/hadoop/app/hadoop-2.6.4
export PATH=${HADOOP_HOME}/bin:${HADOOP_HOME}/sbin:$PATH

#版本1的问题：
#虽然上传到Hadoop集群上了，但是原始文件还在。如何处理？
#日志文件的名称都是xxxx.log1,再次上传文件时，因为hdfs上已经存在了，会报错。如何处理？

#如何解决版本1的问题
#       1、先将需要上传的文件移动到待上传目录
#	2、在讲文件移动到待上传目录时，将文件按照一定的格式重名名
#		/export/software/hadoop.log1   /export/data/click_log/xxxxx_click_log_{date}


#日志文件存放的目录
log_src_dir=/home/hadoop/logs/log/

#待上传文件存放的目录
log_toupload_dir=/home/hadoop/logs/toupload/


#日志文件上传到hdfs的根路径
hdfs_root_dir=/data/clickLog/20151226/

#打印环境变量信息
echo "envs: hadoop_home: $HADOOP_HOME"


echo "log_src_dir:" $log_src_dir
ls $log_src_dir | while read fileName
do
	if [["$fileName" == access.log.*]];
		then
		date = `date + %Y_%m_%d_%H_%M_%S`
		# 将文件移动到待上传目录并重命名
		echo "moving $log_src_dir$fileName to $log_toupload_dir"xxxx_click_log_$fileName"$date"
		mv $log_src_dir$fileName $log_toupload_dir"xxxxx_click_log_$fileName"$date
		# 将待上传文件的path写入到一个列表文件willDoing
		echo $log_toupload_dir"xxxxx_click_log_$fileName"$date >> $log_toupload_dir"willDoing."$date
	fi
		
done
#找到列表文件willDoing
ls $log_toupload_dir | grep will |grep -v "_COPY_" | grep -v "_DONE_" | while read line
do
	#打印信息
	echo "toupload is in file:"$line
	#将待上传文件列表willDoing改名为willDoing_COPY_
	mv $log_toupload_dir$line $log_toupload_dir$line"_COPY_"
	#读列表文件willDoing_COPY_的内容（一个一个的待上传文件名）  ,此处的line 就是列表中的一个待上传文件的path
	cat $log_toupload_dir$line"_COPY_" |while read line
	do
		#打印信息
		echo "puting...$line to hdfs path.....$hdfs_root_dir"
		hadoop fs -put $line $hdfs_root_dir
	done	
	mv $log_toupload_dir$line"_COPY_"  $log_toupload_dir$line"_DONE_"
done



```



