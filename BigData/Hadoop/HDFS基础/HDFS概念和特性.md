# HDFS概念和特性

## 1.重要特性

1. HDFS中的文件在物理上是分块存储（block），块的大小通过配置参数（dfs.blocksize）来规定。
2. HDFS文件系统会给客户端提供一个统一的抽象目录树，客户端通过路径来访问文件。
3. 目录结构及文件分块信息（元数据）的管理由nameNode节点来承担，nameNode是HDFS集群主节点，负责维护整个HDFS文件系统的目录树，以及每一个路径所对应的block块信息（block id及所在的DataNode服务器）
4. 文件的各个block的存储管理由nameNode节点承担，DataNode是HDFS集群从节点，每一个block都可以在多个DataNode上存储多个副本（副本数量也可以通过dfs.replication设置）
5. HDFS是设计成适应一次写入，多次读出的场景，且不支持文件的修改（适合做数据分析，不适合做网盘应用，网络开销大）

## 2.工作机制

1. HDFS集群分为两大角色：nameNode和DataNode
2. NameNode负责管理整个文件系统的元数据
3. DataNode负责管理用户的文件数据块
4. 文件会按照固定的大小（blocksize）切成若干块后分布式存储在若干台DataNode上
5. 每一个文件块可以有多个副本，并存放在不同的DataNode上
6. DataNode会定时向NameNode汇报自身所保存的文件block信息，而nameNode则会负责保持文件的副本数量
7. HDFS的内部工作机制对客户端保持透明，客户端请求访问HDFS都是向nameNode申请来进行

## 3.HDFS写数据流程

客户端向HDFS写数据，首先要跟nameNode通信以确定可以写文件，并获得接收文件block的DataNode，然后客户端按照顺序将各个文件逐个block传递给响应的dataNode，并由接收到block的dataNode负责向其他dataNode幅值block的副本。

### 详细步骤图

![hdfs写数据流程示意图v_shizhan03](.\image\hdfs写数据流程示意图v_shizhan03.png)

1. 根据nameNode通信请求上传文件，nameNode检查目标文件是否已存在，父目录是否存在
2. nameNode返回是否可以上传
3. client请求第一个block应传递到哪些dataNode服务器上
4. nameNode返回3个dataNode服务器ABC
5. client请求3台dn中的一台A上传数据（本质为RPC调用，建立pipeline）,A接收请求后会继续调用B，B调用C，将整个pipeline建立完成，逐级返回应答给客户端。
6. client开始往A上传第一个blok（先从磁盘读取数据放置到一个本地内存缓存），传递的单位为packet（大小64k），先写到缓冲ByteBuf，然后，同时写到文件和socket channel中，A收到一个packet就会传递给B，B传递给C；几乎同时完成传递到三个dataNode。A每传一个packet就会放入到一个应答队列中等待应答。
7. 只要上传成功一台就可以了，nameNode会异步复制，非阻塞
8. 当一个block传输完成后，client再次请求nameNode上传第二个block服务器
9. nameNode记录文件路径、block、副本等信息

## 4.HDFS 读数据流程

客户端将要读取的文件路径发送给nameNode，nameNode获取文件的元数据信息（主要是block的存放位置信息）返回给客户端，客户端根据返回的信息找到对应的dataNode，逐个获取文件的block并在客户端本地进行数据追加合并，从而获得整个文件。

![hdfs读数据流程示意图v_shizhan03](.\image\hdfs读数据流程示意图v_shizhan03.png)

### 详细步骤

1. 跟nameNode通信查询元数据，找到文件块所在的dataNode服务器
2. 挑选一台dataNode（就近原则，然后随机）服务器，建立socket流
3. dataNode开始发送数据（从磁盘中读取数据放入流，以packet为单位来进行校验）
4. 客户端以packet为单位进行接收，先在本地缓存，后写入到文件中

## 5.NameNode工作机制

NameNode的职责：负责客户端请求响应；元数据的管理（查询、修改）

### 元数据管理

nameNode对数据的管理采用三种存储形式：

1. 内存元数据
2. 磁盘元数据镜像文件
3. 数据操作日志文件（可通过日志运算出元数据）

### 元数据存储机制

1. 内存有一份完整的元数据（内存 meta data）
2. 磁盘有一个"准完整"的元数据镜像（fsimage）文件（在nameNode的工作目录中）
3. 用于衔接内存metadata和持久化元数据镜像fsimage之间的操作日志（edtis文件）

**特殊说明：** 当客户端对HDFS中的文件进行新增或者修改操作，操作记录首先被记录到edtis日志文件中，当客户端操作成功后，相应的元数据会更新到内存meta.data中

### 元数据手动查看

可以通过hdfs的一个工具来查看edits中的信息

```Linux
bin/hdfs oev -i edits -o edits.xml
bin/hdfs oiv -i fsimage_0000000000000000087 -p XML -o fsimage.xml
```

### 元数据checkPoint

每隔一段时间，由secondary nameNode将nameNode上积累的edits和一个最先的fsimage下载到本地，并加载到内存进行merge（合并），这个过程称为checkpoint

![secondarynamenode元数据checkpoint机制](.\image\secondarynamenode元数据checkpoint机制.png)



#### 详细过程

nameNode端元数据更新

1/ 元数据更改，更新nameNode的内存

2/记录操作日志（edits）追加操作

合并checkPoint触发条件：定时（两次checkpoint之间的时间间隔3600s）；edits中的记录数量

**过程步骤：**

1. secondary nameNode向nameNode发出请求，是否条件满足，是否需要checkpoint
2. secondary nameNode向nameNode发出checkpoint请求
3. 正在写的是edits叫edits_inprogress，正在写的立即截断(nameNode滚动)，形成刚刚写好的edits
4. 刚刚的edits加上其他的edits以及fsimage一起被secondary nameNode的机器下载下来
5. 加载到内存，合并更新后的元数据
6. dump成新的文件叫fsimage.checkpoint
7. 上传到nameNode机器后，被nameNode重命名为fsimage

注意事项：

第一次下载包括fsimage，后续只下载edits

每一个block占元数据150byte，系统里面存大文件划算，减小nameNode中元数据的大小，可能dataNode没满，dataNode满了。

nameNode 的工作目录应该配置在多个磁盘上，同时往两块磁盘写日志，数据是一样的。

namenode和secondary namenode的工作目录存储结构完全相同，所以，当namenode故障退出需要重新恢复时，可以从secondary namenode的工作目录中将fsimage拷贝到namenode的工作目录，以恢复namenode的元数据

### 元数据目录说明

在第一次部署好Hadoop集群的时候，我们需要在NameNode（NN）节点上格式化磁盘：

```shell
$HADOOP_HOME/bin/hdfs namenode -format
```

格式化完成之后，将会在$dfs.namenode.name.dir/current目录下如下的文件结构

```shell
current/
|-- VERSION
|-- edits_*
|-- fsimage_0000000000008547077
|-- fsimage_0000000000008547077.md5
`-- seen_txid

```

其中的dfs.name.dir是在hdfs-site.xml文件中配置的，默认值如下：

```shell
<property>
  <name>dfs.name.dir</name>
  <value>file://${hadoop.tmp.dir}/dfs/name</value>
</property>

hadoop.tmp.dir是在core-site.xml中配置的，默认值如下
<property>
  <name>hadoop.tmp.dir</name>
  <value>/tmp/hadoop-${user.name}</value>
  <description>A base for other temporary directories.</description>
</property>

```

nameNode 可以配置多个目录，中间使用逗号隔开，各个目录存储的文件结构和内容完全一样，相当于备份，这样的好处在于其中的一个目录损坏了，也不会影响到Hadoop的元数据。

对于$dfs.namenode.name.dir/current/目录下的文件进行解释：

1. VERSION 文件是Java属性文件，其内容大致为

```shell
#Fri Nov 15 19:47:46 CST 2013
namespaceID=934548976
clusterID=CID-cdff7d73-93cd-4783-9399-0a22e6dce196
cTime=0
storageType=NAME_NODE
blockpoolID=BP-893790215-192.168.24.72-1383809616115
layoutVersion=-47

```

（1）namespaceID 是文件系统的唯一标识符，在文件系统首次格式化之后生成的

（2）storageType说明这个文件存储的什么进程的数据结构信息（DataNode， storageType=DATA_NODE）

（3）CTime表示NameNode存储时间的创建时间，每次NameNode进行更新后，都会记录更新时间戳

（4）layoutVersion表示HDFS永久性数据结构的版本信息，只要数据结构变更，版本号也要递减，此时的HDFS也需要升级，否则磁盘仍旧使用的旧版本的数据结构，这会导致新版本的NameNode无法使用。

（5）clusterID是系统或手动指定的集群ID，在-clusterId选项中可以使用它

（6）blockpoolID：是针对每一个Namespace所对应的blockpool的ID，上面的这个BP-893790215-192.168.24.72-1383809616115就是在我的ns1的namespace下的存储块池的ID，这个ID包括了其对应的NameNode节点的ip地址。

**关于clusterID的指定：**

a、使用如下命令格式化一个Namenode：

$HADOOP_HOME/bin/hdfs namenode -format [-clusterId <cluster_id>]

选择一个唯一的cluster_id，并且这个cluster_id不能与环境中其他集群有冲突。如果没有提供cluster_id，则会自动生成一个唯一的ClusterID。

b、使用如下命令格式化其他Namenode：

 $HADOOP_HOME/bin/hdfs namenode -format -clusterId <cluster_id>

c、升级集群至最新版本。在升级过程中需要提供一个ClusterID，例如：

$HADOOP_PREFIX_HOME/bin/hdfs start namenode --config $HADOOP_CONF_DIR  -upgrade -clusterId <cluster_ID>

如果没有提供ClusterID，则会自动生成一个ClusterID。

**关于seen_txid:**

$dfs.namenode.name.dir/current/seen_txid非常重要，是存放transactionId的文件，format之后是0，它代表的是nameNode里面edits_*文件的尾数，nameNode重启的时候，会按照seen_txid的数字，循环从头跑edits_0000001~seen_txid的数字，所以当你的hdfs发生异常重启的时候，一定要对比seen_txid内的数字是不是你edits最后的尾数，不然会发生重建nameNode时metaData的资料有缺失，导致误删DataNode上多余的Block的资讯。 **seen_txid**中记录的是edits滚动的序号，每次重启nameNode时，nameNode就知道要将哪些edits进行加载。

$dfs.namenode.name.dir/current目录下在format的同时，也会生成fsimage和edits文件，及其对应的md5校验文件。

## 6.DataNode的工作机制

### DataNode的工作职责

存储管理用户的文件块数据，定期向nameNode汇报自身所持有的block信息（通过心跳进行上报）

### DataNode掉线判断时限参数

dataNode进程死亡或者网络故障造成dataNode无法与nameNode通信，nameNode不会立即把该节点判定为死亡，要经过一段时间，这段时间暂称为超时时长。HDFS默认的超时时长为10分钟+30秒。如果定义了超时时间timeOut，则超时时长的计算公式为

```mathematica
	timeout  = 2 * heartbeat.recheck.interval + 10 * dfs.heartbeat.interval
```

而默认的heartbeat.recheck.interval 大小为5分钟，dfs.heartbeat.interval默认为3秒

需要注意的是hdfs-site.xml 配置文件中的heartbeat.recheck.interval的单位为毫秒，dfs.heartbeat.interval的单位为秒。所以，举个例子，如果heartbeat.recheck.interval设置为5000（毫秒），dfs.heartbeat.interval设置为3（秒，默认），则总的超时时间为40秒

```xml
<property>
        <name>heartbeat.recheck.interval</name>
        <value>2000</value>
</property>
<property>
        <name>dfs.heartbeat.interval</name>
        <value>1</value>
</property>

```



