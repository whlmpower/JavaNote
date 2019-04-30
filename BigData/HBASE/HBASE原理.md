# HBASE原理

## 1.Region的定义与数据存储管理

![20180625230051678](.\image\20180625230051678.png)

1. 如上t_product表中分为2个列族，数据首先根据rowkey进行划分，每个rowkey在内部根据列族再次细分存储，大部分情况下HBASE存储的数据都是及其庞大的。
2. HBASE内部，根据rowkey的范围将表数据进行划分并存储到region中，每个Region（默认大小10G）都由RegionServer来管理，一个RegionServer可以管理多个Region。
3. 当客户端要对HBASE数据库进行查询的时候，首先找到RegionServer，然后通过RegionServer访问内部的Region。
4. Hmaster在客户端读写操作的时候并不做任何参与，它主要对RegionServer进行管理的。假如某种情况下RegionServer挂掉了，那么Hmaster会将挂掉的RegionServer管理的Region较优其他的RegionServer来管理。
5. 除此之外，Hmaster还有另一个作用，当一个Region的数据不断变大后，HBASE会将一个Region拆分成两个Region，并通过Hmaster将Region分配给新的RegionServer。

## 2.细化HBASE内存存储

![20180625230120392](.\image\20180625230120392.png)



1. 一个Region表示特定范围的rowkey数据集合，而一条rowkey内部以列族的方式存储，这里store描述的是某个rowkey集合的列族存储信息。store逻辑概念。

2. 一个store的容量也是极其庞大的，所以一个store在物理上由多个storeFile组成，storeFile以HFile格式保存在HDFS上。



![20180625230134260](.\image\20180625230134260.png)

首先Hfile文件是不定长的，长度固定的只有两块：Trailer和FileInfo。

Trailer中有指针指向其他数据快的起始点

File Info中记录了文件的一些Meta信息。例如：AVG_KEY_LEN, AVG_VALUE_LEN, LAST_KEY, COMPARATOR, MAX_SEQ_ID_KEY等。

Data Index和Meta Index记录了每块Data块和Meta块的起始点。

Data Block是HBASE I/O的基本单元，为了提高效率，HRegionServer中有基于LRU的Block Cache机制。每个Data块的大小可以在创建一个Table的时候通过参数指定，大号的Block有利于顺序Scan，小号的Block有利于随机查询。每个Data块除了开头的Magic以外就是一个个KeyValue对拼接而成，Magic为随机数，目的是为了防止数据损坏。

3. 客户端在查询表数据的时候会附带row Key ,系统根据RowKey找到对应的RegionServer内部的Region，但是一个Region内部还存在着多个Store，Store内部还有StoreFile，如何确定找到StoreFile内存查询数据，依赖于HBASE表内部的布隆过滤器。

![20180625230149132](.\image\20180625230149132.png)

4. 当一条数据插入到StoreFile的时候，首先会将rowkey映射成为64bit的二进制数据，并存储在storeFile对应的bit数组中。
5. 当执行查询的时候，根据rowkey的范围线确定好查询的Region，在Region内部找到对应的列族Store，并遍历Store内部所有的StoreFile对应的数组，如果找到的rowkey与匹配的rowkey与匹配的rowkey一致，则读取该storeFile内部的数据。

## 3.HBASE内存管理与文件合并

![20160501215448050](.\image\20160501215448050.png)

1. 当客户端写入数据的时候，首先会将数据直接写入到MEMStore中，当MEMStore达到了HBase.hRegion.MemStore.flush.size 限定的值，就会将数据刷新到一个新的StoreFile，但是假如在这个过程中，MEMStore所在的RegionServer挂掉了，就必须借助于Hlog进行数据的恢复。

WAL（Write ahead log）,用来做灾难恢复的，每一个RegionServer维护一个Hlog。

当客户端往HregionServer中写数据的时候，除了向HRegionServer发送写操作，还会降当前的操作异步写入到Hlog中，而Hlog中存储的数据也是持久化到HDFS中的。

![1123009-20170313151839198-1335459231](.\image\1123009-20170313151839198-1335459231.png)

2. HBASE的数据实际存储在HDFS中的，而HDFS中的数据也是不支持修改的。

比如我们在客户单执行了一个删除操作，当再次查询数据的时候被干掉了。而在底层，其实只是对被删除的数据做了一个标记，HDFS上的数据并未被删除掉。

比如多次修改操作，可以查询到数据库中显示的数据被改变了，其实在HDFS上也并未对数据进行修改。当执行更新后，原来的数据其实还在HDFS上。

3. HBASE为了防止小文件（被刷到磁盘的MEMStore）过多，以保证效率，HBASE需要在必要的时候将这些小的StoreFile合并成较大的StoreFile，这个过程称为Compaction，在HBASE中，主要存在两种Compaction:Minor Compaction 和 Major Compaction。

minorCompaction选取一些小的、临近的StoreFile，将他们重写为一个大的StoreFile，出于性能的考虑，minor Compaction不会删除过期的或者标记为要删除的数据。minor Compaction的结果获得更少的更大的StoreFile。

major Compaction的结果是每个StoreFile生成一个StoreFile。同时在生成的过程中会删除已经标记要删除的数据。majorCompaction理论上会提高性能。然而，在一个高负载的系统中，major Compaction会导致一些不利的影响。

### 主要角色总结

Client：包含HBASE的接口，Client维护着一些Cache来加快对HBASE的访问，比如Region的位置信息。

Master：HBASE Master用于协调多个Region Server，侦测各个RegionServer之间的状态，并平衡RegionServer之间的负载。HBASEMaster还有一个职责便是负责分配Region给RegionServer。HBASE允许多个Master节点共存，但是这需要Zookeeper的帮助。不过当多个Master节点共存时，只有一个Master是提供服务的，其他的Master节点处于待命的状态。当正在工作的Master节点宕机，其他的Master则会接管HBASE的集群。

1 为RegionServer分配Region

2 负责RegionServer的负载均衡

3 发现失效的Region Server并重新分配其上的Region

4 HDFS上的垃圾回收

5 处理Schema的更新请求



Region Server：对于一个RegionServer而言，其包括了多个Region。RegionServer的作用只是管理表格，以及实现读写操作。Client直接连接RegionServer，并通信获取HBASE中的数据。对于Region而言，则是真实存放HBASE数据的地方，也就是Region是HBASE可用性和分布式的基本单位。如果当一个表格很大，并由很多歌CF组成时，那么表的数据将存放在多个Region之间，并且在每个Region会关联多个存储的单元（Store）。

1 RegionServer维护Master分配给他的Region，处理这些Region的IO请求

2 RegionServer负责切分在运行过程中变得更大的Region

client访问HBASE上的数据的过程并不需要master参与（寻址访问Zookeeper和RegionServer，数据访问读写RegionServer），master仅仅维护着table和Region的元数据信息，负载很低。



Zookeeper：对于HBASE而言，Zookeeper的作用是至关重要的。首先Zookeeper是作为HBASE Master的HA的解决方案。也就是说，是Zookeeper保证了至少有一个HBASE Master处于运行状态。并且Zookeeper负责Region和RegionServer的注册。

1 保证任何时候，集群中只有一个master

2 存储所有Region的寻址入口，root表在哪台服务器上

3 实时监控RegionServer的状态，将RegionServer的上线和下线通知实时通知给Master

4 存储HBASE的schema，包括有哪些table，每个table有哪些column family

### 基本存储的总结

1. Table 中所有行都按照rowkey的字典序排列
2. Table在行的方向上分割为多个Hregion
3. Region按照大小进行分割，每个表一开始只有一个Region，随着数据不断插入表，Region不断增大，当增大到一个阈值的时候，Hregion就会等分为两个新的Region。当table中的行不断增多，就会有越来越多的Hregion。
4. Hregion是HBASE中分布式存储和负载均衡的最小单元。最小单元就表示不同的Hregion可以分布在不同的HRegion server上，但一个Hregion是不会拆分到多个Server上的
5. Hregion虽然是负载均衡的最小单元，但不是物理存储的最小单元。事实上，Hregion由一个或者多个Store组成，每个Store存储一个column family。每个Store又由一个MEMStore和0个至多个StoreFile组成。
6. 一个Region由多个Store组成，每个Store包含一个列族所有数据；Store包括位于内存的MEMStore和位于硬盘的StoreFile

写操作先写入到MEMStore，当MEMStore中的数据量达到某个阈值，HRegionServer启动FlashCache进程写入到StoreFile，每次写入形成单独的一个StoreFile；当StoreFile大小超过一定阈值后，会将当前的Region分割成两个，并由Hmaster分配给响应的Region服务器，以实现负载均衡。

客户端检索数据时，先在MEMStore中查找，找不到再查找StoreFile。

## 4.HBASE客户端搜索RegionServer



![20180625230250243](.\image\20180625230250243.png)

首先HBASE内部内置了一张.META表，该表记录了用户表的每个Region信息，重要的两行分别是Row Key和 Server。对于Row Key内部需要存储用户表tableName，用户每个Region 的Start Kay，时间戳。对于Server则是存储该Region存放在哪个RegionServer中。

客户端在查询、删除、插入的时候，首先通过.Meta表查找到对应的RegionServer，在该RegionServer中遍历Region。

上述过程中存在一个新的问题，.Meta表本身就是一个HBASE表，要查询表中的数据需要知道存储该数据的RegionServer存放在哪里，为了解决这个问题，引入了Zookeeper，将所有的RegionServer存储到Zookeeper中。

新的问题2，当table1表的内容变得越来越大的情况下，记录该表的.Meta表的内容也随着正大，随着Meta表内容变大，所需要的RegionServer（Region变大）也会随着变大。也就是查询一条数据，在Zookeeper中会有越来越多的RegionServer要逐一进行遍历。为了解决这个问题，引入了-Root-的表。

![20180625230301504](.\image\20180625230301504.png)

-root-表只会有一个Region，这个Region的信息也是存在HBASE内部的，完整的步骤：

1. Client假设要查找表2，发出命令 get 'table2'，‘RK500’；首先会去Zookeeper中查找Root表唯一的RegionServer。并匹配到META.table2.RK0,RK20000,timestamp的Row key，找到对应的RS20(RegionServer)。
2. 找到RegionServer 20  后，会根据row key”RK500”匹配到 table2.RK0.timestamp的RegionServer 1。
3. 在RegionServer中根据bloomFiter找到对应的StoreFile并查询数据。



![1123009-20170313151639698-1448537420](.\image\1123009-20170313151639698-1448537420.png)

**为了提高效率，Meta表中的数据会存储到内存中；client会将查询过的位置信息缓存起来，缓存不会主动失效**

如果客户端根据缓存信息还访问不到数据，则询问相关.META.表的Region服务器，试图获取数据的位置，如果还是失败，则询问-ROOT-表相关的.META.表在哪里。最后，如果前面的信息全部失效，则通过ZooKeeper重新定位Region的信息。所以如果客户端上的缓存全部是失效，则需要进行6次网络来回，才能定位到正确的Region。

## 5.Hbase 读写过程

HBASE使用MEMStore和StoreFile存储对表的更新，数据在更新的时候首先写入到Hlog和MEMStore。MEMStore中的数据是排序的，当MEMStore中的累积到一定的阈值的时候，会创建一个新的MEMStore，并将老的MEMStore添加到Flush队列，由单独的线程Flush到磁盘中，成为一个StoreFile。与此同时，系统会在Zookeeper中记录一个CheckPoint，表示这个时刻之前的数据源变更已经持久化了。当系统出现意外时，可能导致MEMStore中的数据的丢失，此时可以使用Hlog来恢复Checkpoint之后的数据。StoreFile是只读的，一旦创建后就不能进行再次修改。因此HBASE的更新是不断的追加操作。当一个Store中的StoreFile达到一定的阈值后，就会进行一次合并操作，将会对同一个key的修改合并到一起，形成一个大的StoreFile。当StoreFile的大小超过一定阈值后，又会对StoreFile进行切分操作，等分为两个StoreFile。

### 写操作流程

1. Client通过Zookeeper的调度，向RegionServer发出写数据请求，在Region中写数据
2. 数据被写入到Region的MEMStore中，直到MEMStore达到预设阈值
3. MEMStore中的数据被Flush成一个Stor
4. 随着StoreFile文件不断增多，其数量增长到一定阈值后，触发Compact合并操作，将多个StoreFile合并成为一个StoreFile，同时进行版本合并和数据删除
5. StoreFile通过不断的Compact合并操作，逐步形成越来越大的StoreFile。
6. 单个StoreFile的大小超过一定阈值后，触发Split操作，把当前的Region Split成2个新的Region。父Region会下线，新的Split出的2个子Region会被HMaster分配到相应的RegionServer上，使得原先一个Region的压力分流，实现负载均衡

可以看出HBase只有增添数据，所有的更新和删除操作都是在后续的Compact历程中举行的，使得用户的写操作只要进入内存就可以立刻返回，实现了HBase I/O的高机能

### 读操作流程

1. Client访问Zookeeper，查找-Root-表，获取.Meta表信息
2. 从.Meta表查找，获取存放目标数据的Region信息，从而找到对应的RegionServer
3. 通过RegionServer找到需要查询的数据
4. RegionServer的内存分为MEMStore和BlockCache两个部分，MEMStore主要用于写数据，BlockCache主要用于读数据。读请求先到MEMStore中查数据，查不到就到BlockCache中查，查不到就到StoreFile上读，并把读的结果放入到BlockCache

寻址过程：client-->Zookeeper-->-ROOT-表-->.META.表-->RegionServer-->Region-->client