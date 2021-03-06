# Elasticsearch 权威指南（搜索章节）

## 1.搜索——基本的工具

A search can be :

1. 在类似gender或age这样字段上使用结构化查询，join_date这样的字段上使用排序
2. 全文检索，使用所有字段匹配关键字，按照关联性排序返回结果
3. 结合以上两条

三个概念：

**映射（Mapping）**：数据在每个字段中的解释说明

**分析（Analysis）**：全文是如何处理的，可以被搜索的

**领域特定语言查询（Query DSL）**：Elasticsearch使用的灵活的、强大的查询语言

## 2.空搜索

```JSON 
GET /_search
```

响应中的hits，包含了total字段来表示匹配到的文档总数。

每个节点都有 _score 字段，相关性得分，衡量与文档匹配程度。

took 花费毫秒数。

shards 参与查询的分片数。

timeout 超时与否，超时并非熔断器

## 3.多索引和多类别

通过限制搜索的不同索引或类型，我们可以在集群中跨所有文档搜索。Elasticsearch转发搜索请求到集群中平行的主分片或每个分片的复制分片上，收集结果后选择顶部十个返回给我们

**搜索一个索引有5个主分片和5个索引各有一个分片事实上是一样的**

## 4.分页

Elasticsearch接受from和size参数：

size： 结果数，默认为10

from：跳过开始的结果数，默认为0

当心分页太深或者一次请求太多的结果，每个分片生成自己排好序的几个，集中起来排序以确保整体排序正确。

## 5.简易搜索

search API两种表单：查询字符串和DSL

查询字符串搜索对于在命令行下运行点对点查询特别有用。例子：类型为tweet并在tweet字段中包含Elasticsearch字符的文档

```JSON 
GET /_all/tweet/_search?q=tweet:elasticsearch
```

**_all 字段**

当你索引一个文档，Elasticsearch把所有字符串字段值连接起来放在一个大字符串中，被索引为一个特殊的字段 _all 

若没有指定字段，查询字符串搜索，(q=xxx)使用_all 字段进行搜索。

