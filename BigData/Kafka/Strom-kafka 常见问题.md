# Strom-kafka 常见问题

## 1. Kafka+Storm如何保证消息完整处理

一条消息的产生--->KafKa--->KafkaSpout-Storm---->Redis

### KafKa数据生产消费如何保证消息的完整处理

Producer-batch（同步/异步（缓存机制queue 缓冲区的时间阈值和数量阈值））--重试机制--->ack(-1,1,0)--->Broker（partition的目录）----->ZK（offset）----->KafkaSpout（ack fail （重发机制，需结合自定义Map或者外部数据存储（Redis）完成数据暂存））----->bolt（订单ID，去重（结合Redis set数据结构））

Producer发送时缓存数据，需要阈值设定（数量阈值，时间阈值），当数据太多并来不及发送时，会产生老数据是否保留的问题，如何配置文件中配置的是删除掉，数据就丢失了。

**Spout ack 处理后，去ZK上更新offset**

**消息重复消费问题**：ZK上的offset=10000；一批一批地拉取（10000 + num）,定时更新到ZK上（依据时间阈值和数量阈值触发），更新offset 10000 + num，在此时Kafka-spout失败了，会导致重复消费问题。

KafkaConsumer 消费数据时，由于offset是周期性更新，导致zk上的offset值必然小于Kafkaconsumer内存中的值，当KafkaConsumer挂掉后重启，必然会导致数据重复消费。
		如何解决重复消费：1，技术方法(不可取)，2，业务方法（标识数据：找到消息中的唯一标识 <messageTag,isProcess>），利用Redis Set等相关数据结构完成消息去重。

### 问题2：Storm中如何保证消息的完整处理  ack - fail 
​		 自己定义 缓存Map<msgid,messageObj>
​		 Spout---->nextTuple----Tuple---Bolt(ack(tuple))
​										Bolt1-----Tuple1-1
​										Bolt1-----Tuple1-2
​										Bolt1-----Tuple1-3
​										Bolt1-----Tuple1-4
​													Bolt2(ack(Tuple1-1))
​													Bolt2(ack(Tuple1-2))
​													Bolt2(ack(Tuple1-3))
​													Bolt2(ack(Tuple1-4))
​													

		 如果成功（spout.ack(msg)）
			map.remove(msgid)
		 如果失败（spout.fail(msg)）
			messageObj = map.get(msgid)
			collector.emit(messageObj,msgid)
	

### 问题3：如何保证一条消息路过各个组件时保证全局的完整处理
		保存每个环节的数据不丢失，自然就全局不丢失。

### 问题4：在storm环节中自定义的map如何存储，在KafkaConsumer如何处理重复消费的问题？

		
			1，保存在当前Jvm中，既然出异常导致消息不能完全处理，存放在jvm中的标识数据，缓存Map必然会丢失
			2，保存在外部的存储中，redis。  标识数据<Set>  缓存Map<Map> 
				问题：请问存储redis时，由于网络原因或其他异常导致数据不能成功存储 怎么办？
				重试机制，保存3次，打印log日志，redis存储失败。
## 数据量大如何保证到Kakfa中，storm如何消费，解决延迟。
​		问题1：请问Kafka如何处理海量数据       -----整体数据：100G
​			  数据从何而来：producer 集群----DefaultPartition  ------保存数据平均分配
​			  

			  数据保存在哪里： broker 集群 
								topic ---partition  10   -------------->每个分片保存10G 
			  数据保存如何快：pageCache 600M/S的磁盘写入速度  sendfile技术
	
		问题2: 请问Storm如何处理海量数据，尽可能快
			  数据输入： KafkaSpout(consumerGroup,10) 读取外部数据源
			  数据计算： 数据计算是根据对数据处理的业务复杂度来的，越复杂并发度越大。
			             如果bolt的并发读要设置成1万个，才能提高处理速度，很显然是不行的。需要将bolt中的代码逻辑分解出来，形成多个bolt组合。
						 bolt1--->bolt2--->bolt3......			 
						 
	
## Flume监控文件，异常之后重新启动，如何避免重复读取
   异常的范围是 flumeNg的异常
   log文件，读取文件，保存记录的行号
   问题1：如何保存行号
		command.type = exec shell<tail -F  xxx.log>  自定义shell脚本，保存消费的行号到工作目录的某个文件里。
		当下次启动时，从行号记录文件中拿取上次读到多少行。