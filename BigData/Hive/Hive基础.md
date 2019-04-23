# Hive的基础整理

## 1. Hive的数据存储

1. Hive中所有的数据都存储在HDFS中，没有专门的数据存储格式（Text，SequenceFile，ParquetFile，RCFILE等）

2. 只需要在创建表的时候告诉Hive数据中的列分隔符和行分隔符，Hive就可以解析数据

3. Hive中包含以下数据模型：DB，Table，External Table，Partition，Bucket

   db：在hdfs中表现为${hive.metastore.warehouse.dir}目录下一个文件夹

   table：在hdfs中表现所属db目录下一个文件夹

   external table：外部表, 与table类似，不过其数据存放位置可以在任意指定路径

   普通表: 删除表后, hdfs上的文件都删了

   External外部表删除后, hdfs上的文件没有删除, 只是把文件删除了

   partition：在hdfs中表现为table目录下的子目录

   bucket：桶, 在hdfs中表现为同一个表目录下根据hash散列之后的多个文件, 会根据不同的文件把数据放到不同的文件中

## 2.Hive的基本操作

### DDL操作

#### 创建表

```sql
CREATE [EXTERNAL] TABLE [IF NOT EXISTS] table_name 
   [(col_name data_type [COMMENT col_comment], ...)] 
   [COMMENT table_comment] 
   [PARTITIONED BY (col_name data_type [COMMENT col_comment], ...)] 
   [CLUSTERED BY (col_name, col_name, ...) 
   [SORTED BY (col_name [ASC|DESC], ...)] INTO num_buckets BUCKETS] 
   [ROW FORMAT row_format] 
   [STORED AS file_format] 
   [LOCATION hdfs_path]

```

**说明：**

1. CREATE TABLE 创建一个指定名字的表。如果相同名字的表已经存在，则抛出异常；用户可以用
   IF NOT EXISTS 选项来忽略这个异常。

2. External 可以让用户创建一个外部表，在建表的同时指定一个指向实际数据的路径。Hive创建内部表的时候，会将数据移动到数据仓库指向的路径。若创建外部表，仅记录数据所在的路径，不对数据的位置做任何改变。在删除表的时候，内部表的元数据和数据会被一起删除，而外部表只删除元数据，不删除数据。

3. LIKE 允许用户复制现有的表结构，但是不复制数据。

4. ROW FORMAT DELIMITED     （ row format 指 行和一行中的字段如何存储）

   [FIELDS TERMINATED BY char] 

   [COLLECTION ITEMS TERMINATED BY char] 

   ​     [MAP KEYS TERMINATED BY char] [LINES TERMINATED BY char] 

      | SERDE serde_name [WITH SERDEPROPERTIES (property_name=property_value, property_name=property_value, ...)]

   用户在创建表的时候可以自定义SerDe或者使用自带的SerDe。如果没有指定 ROW
   FORMAT 或者 ROW FORMAT DELIMITED，将会使用自带的 SerDe。在建表的时候，用户还需要为表指定列，用户在指定表的列的同时也会指定自定义的 SerDe，Hive通过 SerDe 确定表的具体的列的数据。

5. STORED AS 

   SEQUENCEFILE|TEXTFILE|RCFILE

   如果文件数据是纯文本，可以使用 STORED AS TEXTFILE。如果数据需要压缩，使用 STORED AS SEQUENCEFILE。

6. Clustered by

   对于每一个表（table）或者分区，Hive可以进一步组织成为桶，桶是更为细粒度的数据范围划分。Hive也是针对某一列进行桶的组织。Hive采用对列hash，然后除以桶的个数求余的方式决定了该条记录存放在哪个桶当中。

   