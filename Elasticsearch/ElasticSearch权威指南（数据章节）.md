# ElasticSearch权威指南

## 数据

Elasticsearch 是一个分布式文档存储引擎，每一个字段都是默认被索引的，每个字段专门有一个反向索引用于快速检索。

### 1. 文档

特指最顶层结构或者根对象序列化成JSON数据(以唯一ID标识并存储于ElasticSearch中)

**文档元数据：**

_index : 文档存储的地方（类似于关系型数据库中的"数据库"）

_type：文档代表对象的类（表）（使用相同的type的文档表示相同的事物）（每个type都有自己的映射或者结构定义，类似于传统数据库中的列）

_id：文档的唯一标识（与 _index 和 _type 组合时，就可以在ElasticSearch中唯一标识一个文档）

### 2.索引一个文档

_id 自定义与自动生成

每个文档都有版本号，每当文档变化时，都会使 _version增加

**自增ID**

让ElasticSearch 自动生成，请求结构发生了变化：PUT（在这个URL中存储文档）变成了POST（在这个type下存储文档）--（原来是把文档存储到某个ID对应的空间，现在是把文档添加到某个_type）

### 3.检索文档

_source字段：包含了在创建索引时我们发送给Elasticsearch的原始文档

found：true表示已找到；false表示没有找到

**curl -i**  参数可以得到相应头

**检索文档一部分**

```html
GET  /website/blog/123?_source=title,text


```

**检索文档是否存在**

使用HEAD方法来代替GET，存在返回200 OK，不存在返回400 Not Found

**更新整个文档**

index API重建索引(reindex)或者替换掉它

### 4.创建新文档

使用op_type 查询参数：

```html
PUT /website/blog/123?op_type=create
{}
```

成功创建新的文档，返回201 created

如果包含相同的_index , _type 和 _id的文档已经存在 将返回409 Conflict

### 5.删除文档

```HTML
DELETE /website/blog/123
```

文档不存在，不会返回404，返回 found字段：false，  _version字段增加

### 6.处理冲突

最近的索引请求会生效，只存储最后被索引的任何文档

并发更新时修改不丢失：**乐观并发控制**

当文档被创建、更新或删除，文档的新版本会被复制到集群的其他节点。Elasticsearch既是同步的又是异步的。_version 保证所有修改都是被正确的排序，当一个旧版本出现在新版本之后，它会被简单的忽略。

```HTML
PUT /website/blog/1?version=1
{
}
```

上述我们只希望文档的 _version是1时更新才生效

所有的更新和删除文档的请求都接受version参数，它可以允许在你代码中增加乐观锁控制

**使用外部版本控制系统**

一种常见的结构是使用一些其他的数据库做主数据库，使用Elasticsearch搜索数据，主数据库发生变化，就要拷贝到Elasticsearch中。如果主数据库有版本字段，类似于timestamp，就可以在Elasticsearch的查询字符串后面添加version_type=external来使用这些版本号。检查是否小于指定的版本，如果请求成功，外部版本就会被存储到 _version 中

```HTML
PUT /website/blog/2?version=5&version_type=external
{}
```

### 7.文档局部更新

**update 来实现局部更新**

```Jso
Post /website/blog/1/_update
{
    "doc":{
        "tags":["testing"],
        "views":0
    }
}
```

**使用脚本局部更新**

```JSON
POST /website/blog/1/_update
{
    "script":"ctx._source.views+=1"
}
```

在tags中添加search

```JSON
POST /website/blog/1/_update
{
    "script":"ctx._source.tags+=new_tag",
    "params":{
        "new_tag":"search"
    }
}
```

通过设置ctx.op 为 delete 根据内容删除文档

```JSON
POST /website/blog/1/_update
{
    "script":"ctx.op = ctx._source.views == count ? 'delete' : 'none'",
    "params":{
        "count":1
    }
}
```

**更新可能不存在的文档**

使用upsert参数定义文档来使其不存在时被创建

```JSON
POST /website/pageviews/1/_update
{
	"script" : "ctx._source.views+=1",
	"upsert": {
		"views": 1
	}
}
```

**更新和冲突**



update API在检索文档的当前_version，重建索引阶段通过index请求提交。其他进程在检索和重建索引阶段修改了文档，  _version 将不能被匹配，然后更新失败

通过retry_on_confict 参数设置重试次数来自动完成。

### 8.检索多个文档

更快的方式是在一个请求中使用multi-get 或者mget API

如果你想检索的文档在同一个 _index 中，在URL中定义一个默认的 / _index 

```JSON
POST /website/blog/_mget
{
    "ids":["2", "1"]
}
```

通过简单的ids 数值来代替完整的docs数组

### 9.更新时的批量操作

mget允许我们一次性检索多个文档，bulk API允许我们使用单一的请求来实现多个文档的create、index、update、或者 delete

格式类似于使用"\n"符号连接起来的一行一行的JSON文档流

```JSON
POST /_bulk
{ "delete": { "_index": "website", "_type": "blog", "_id": "123" }} 
{ "create": { "_index": "website", "_type": "blog", "_id": "123" }}
{ "title": "My first blog post" }
{ "index": { "_index": "website", "_type": "blog" }}
{ "title": "My second blog post" }
{ "update": { "_index": "website", "_type": "blog", "_id": "123", "_retry_on_conflict" :
{ "doc" : {"title" : "My updated blog post"} }
```

bulk 请求不是原子操作，不能实现事务，每个请求操作时都是分开的

**不要重复**

像mget API，bulk请求也可以在URL中使用 / _index 或 / _index/ _type 

```JSON
POST /website/_bulk
{ "index": { "_type": "log" }}
{ "event": "User logged in" }
```

你依旧可以覆盖元数据行的 _index 和 _type ，在没有覆盖时它会使用URL中的值作为默认
值

**多大才算大**

试着批量索引标准的文档，随着大小的增长，当性能开始降低，说明你每个批次的大小太大
了。 

