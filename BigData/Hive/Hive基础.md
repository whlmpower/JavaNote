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
   
   将表(或者是分区)组织成桶有两个理由：
   
   (1)获得更高的查询处理效率。桶为表加上了额外的结构，Hive在处理有些查询时能够利用这个机构。具体而言，连接两个在(包含连接列)相同列上划分了桶的表，可以使用Map端连接（Map-side join）高效的表现。Join操作时，对于Join操作两个表有一个相同的列，如果对这两个表都进行了桶操作。那么将保存相同列值的桶进行Join操作就可以，可以大大减少Join的数据量。
   
   (2)使取样（sampling）更高效。在处理大规模数据集的时候，在开发和修改查询的阶段，如果能在数据集的一小部分数据上试运行查询，就会带来很大的方便。

#### 修改表

**增加删除分区**



**重命名表**

ALTER TABLE table_name RENAME TO new_table_name

**增加更新列**

ADD是代表新增一字段，字段位置在所有列后面（partition 列前面）,Replace 则是表示替换表中所有字段。

**显示命令**

show tables

show databases

show partitions

show functions

desc extended t_name;

desc formatted table_name;

### DDL操作

**Load**

Load 操作只是单纯的复制/移动操作，将数据文件移动到 Hive 表对应的位置。

Local：如果指定了 LOCAL， load 命令会去查找本地文件系统中的 filepath；如果没有指定 LOCAL 关键字，则根据inpath中的uri[[M1\]](#_msocom_1) 查找文件

------



如果指定了 LOCAL，那么： 

load 命令会去查找本地文件系统中的 filepath。如果发现是相对路径，则路径会被解释为相对于当前用户的当前路径。 

load 命令会将 filepath中的文件复制到目标文件系统中。目标文件系统由表的位置属性决定。被复制的数据文件移动到表的数据对应的位置。

 

如果没有指定 LOCAL 关键字，如果 filepath 指向的是一个完整的 URI，hive 会直接使用这个 URI。 否则：如果没有指定 schema 或者 authority，Hive 会使用在 hadoop 配置文件中定义的 schema 和 authority，fs.default.name 指定了 Namenode 的 URI。 

如果路径不是绝对的，Hive 相对于/user/进行解释。 

Hive 会将 filepath 中指定的文件内容移动到 table （或者 partition）所指定的路径中。

**OverWrite 关键字**

如果使用了 OVERWRITE 关键字，则目标表（或者分区）中的内容会被删除，然后再将 filepath 指向的文件/目录中的内容添加到表/分区中。 

如果目标表（分区）已经有一个文件，并且文件名和 filepath 中的文件名冲突，那么现有的文件会被新文件所替代。 

**INSERT**

将查询结果插入到Hive表

导出表数据

1. 导出文件到本地
2. 导出数据到HDFS

**SELECT**

基本SELECT操作

```sql
SELECT [ALL | DISTINCT] select_expr, select_expr, ... 
FROM table_reference
[WHERE where_condition] 
[GROUP BY col_list [HAVING condition]] 
[CLUSTER BY col_list 
  | [DISTRIBUTE BY col_list] [SORT BY| ORDER BY col_list] 
] 
[LIMIT number]

```

1. Order by 会对输入全局做排序，因此只有一个Reducer，会导致当输入规模较大时，需要较长的计算时间，一般使用limit进行短路
2. Sort By不是全局排序，其在数据进入到Reducer前完成排序。因此，如果用sort by进行排序，并且设置mapred.reduce.task>1，则sort by只保证每个reducer的输出有序，不保证全局有序。
3. distribute by根据distribute by指定的内容将数据分到同一个reducer
4. Cluster by除了具有Distribute By的功能外，还会对该字段进行排序，因此，常常任务cluster by = distribute by + sort by

### Hive Join

```sql
join_table:
  table_reference JOIN table_factor [join_condition]
  | table_reference {LEFT|RIGHT|FULL} [OUTER] JOIN table_reference join_condition
  | table_reference LEFT SEMI JOIN table_reference join_condition

```

Hive
支持等值连接（equality
joins）、外连接（outer joins）和（left/right
joins）。Hive **不支持非等值的连接**，因为非等值连接非常难转化到 map/reduce 任务。

1. 只支持等值join
2. 可以join多于两个表

```sql
SELECT a.val, b.val, c.val FROM a JOIN b
    ON (a.key = b.key1) JOIN c ON (c.key = b.key2)

```

如果join中多个表的 join key 是同一个，则 join 会被转化为单个 map/reduce 任务，例如：

```sql
SELECT a.val, b.val, c.val FROM a JOIN b
    ON (a.key = b.key1) JOIN c
    ON (c.key = b.key1)

```

被转化为单个 map/reduce 任务，因为 join 中只使用了 b.key1 作为 join key。

```sql
SELECT a.val, b.val, c.val FROM a JOIN b ON (a.key = b.key1)
  JOIN c ON (c.key = b.key2)

```

而这一 join 被转化为 2 个 map/reduce 任务。因为 b.key1 用于第一次 join 条件，而 b.key2 用于第二次 join。

3. join时，每次map/reduce任务的逻辑：

reducer会缓存join序列中除了最后一个表的所有表的记录，再通过最后一个表将结果序列化到文件系统中。这一实现有助于在reduce端减少内存的使用量。实践中，应该把最大的那个表写在最后。

```sql
 SELECT a.val, b.val, c.val FROM a
    JOIN b ON (a.key = b.key1) JOIN c ON (c.key = b.key1)

```

所有表都使用同一个 join key（使用 1 次 map/reduce 任务计算）。Reduce 端会缓存 a 表和 b 表的记录，然后每次取得一个 c 表的记录就计算一次 join 结果，类似的还有：

```sql
SELECT a.val, b.val, c.val FROM a
    JOIN b ON (a.key = b.key1) JOIN c ON (c.key = b.key2)

```

这里用了 2 次 map/reduce 任务。第一次缓存 a 表，用 b 表序列化；第二次缓存第一次 map/reduce 任务的结果，然后用 c 表序列化。

**Join 发生在Where子句之前**：如果你想限制join的输出，应该在Where子句中写过滤条件，或者是在join子句中写。

**混淆问题**

```sql
SELECT a.val, b.val FROM a
  LEFT OUTER JOIN b ON (a.key=b.key)
  WHERE a.ds='2009-07-07' AND b.ds='2009-07-07'

```

会 join a 表到 b 表（OUTER JOIN），列出 a.val 和 b.val 的记录。WHERE 从句中可以使用其他列作为过滤条件。但是，如前所述，如果 b 表中找不到对应 a 表的记录，b 表的所有列都会列出 NULL，**包括 ds 列**。也就是说，join 会过滤 b 表中不能找到匹配 a 表 join key 的所有记录。这样的话，LEFT OUTER 就使得查询结果与 WHERE 子句无关了。解决的办法是在 OUTER JOIN 时使用以下语法：

```sql
SELECT a.val, b.val FROM a LEFT OUTER JOIN b
  ON (a.key=b.key AND
      b.ds='2009-07-07' AND
      a.ds='2009-07-07')

```

这一查询的结果是预先在 join 阶段过滤过的，所以不会存在上述问题。这一逻辑也可以应用于 RIGHT 和 FULL 类型的join 中。

**left semi join 是IN/Exists的高效实现**



