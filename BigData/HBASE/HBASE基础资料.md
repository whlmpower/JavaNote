# HBASE基础资料

HBASE 是一个面向列、可伸缩（列是可以进行增删的）分布式存储系统。

## 与传统数据库对比

HBASE的优势：

1. 线性扩展，随着数据量增多可以通过节点扩展进行支持
2. 数据存储在hdfs上，备份机制健全
3. 通过Zookeeper协调查找数据，访问数据快。

## HBASE集群中的角色

1. 一个或者多个主节点，Hmaster
2. 多个从节点 HregionServer

## HBASE中的数据模型

行键  时间戳  列族  （列族包括列数据）

与nosql数据库相似，row key是用来检索记录的主键。访问HBASE table中的行，只有三种方式：

1. 通过单个row key 访问
2. 通过row key 的range（正则）
3. 全表扫描

Row Key 行键可以是任意字符串（最大长度是64KB，实际应用中的长度一般为10 -100bytes），在HBASE内部，row key 保存为字节数组。存储时，数据按照Row key的字典序（byte order）排序存储。设计key时，要充分利用排序存储这个特性，经常将一起读取的行存储放到一起。

### Columns Family

列族：HBASE表中每个列，都归属于某个列族，列族是表的schema的一部分（而列不是），必须在使用表之前定义。列名都是以列族作为前缀。

### Cell

由{row key, columnFamily, version} 唯一确定的单元。cell中的数据是没有类型的，全部是字节码形式存贮。

 ### Time Stamp

HBASE 中通过rowkey和columns确定的为一个存贮单元称为cell。每个cell都保存着同一份数据的多个版本(即更新数据数据)。版本通过时间戳来索引。时间戳的类型是 64位整型。

HBASE提供了两种数据版本回收方式。一是保存数据的最后n个版本，二是保存最近一段时间内的版本（比如最近七天）。