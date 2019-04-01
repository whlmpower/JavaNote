# Elasticsearch权威指南（排序章节）

## 1.排序

默认情况下，结果集会按照相关性进行排序，相关性越高，排名越靠前。

### 排序方式

过滤语句与_score 没有关系，但是有隐含的查询条件match_all 为所有的文档的 _score设置为1，也就是相当于所有文档相关性是相同的。

### 字段值排序

按照时间进行排序，使用sort参数进行排序：

```Json
GET /_search
{
    "query":{
        "filtered":{
            "filter":{"term":{"user_id" : 1}}
        }
    },
    "sort":{"date":{"order":"desc"}}
}
```

返回值中， _score 字段没有经过计算，因为没有用作排序。 date 字段被转为毫秒当做排序的依据。

_score 和 max_score 字段都为null，不是相关性排序的时候，就不需要统计其相关性。

### 默认排序

只指定要排序字段的名称，字段值默认以顺序排列，_score 默认以倒序排列。

### 多级排序

合并查询语句，第一排序是date，第二排序是_score

```JSON
GET /_search
{
	"query" : {
		"filtered" : {
			"query": { "match": { "tweet": "manage text search" }},
			"filter" : { "term" : { "user_id" : 2 }}
			}
		},
	"sort": [
		{ "date": { "order": "desc" }},
		{ "_score": { "order": "desc" }}
	]
}
```

多级排序并不需要包含_score，可以自定义数值或字段

### 字符串参数排序

```JSON
GET /_search?sort=date:desc&sort=_score&q=search

```

### 多值字段排序

对于数字和日期，可以从多个值中去除一个来进行排序，可以使用min，max, avg或sum这些模式

在date字段中用最早的日期进行排序

```JSON
"sort": {
	"dates": {
		"order": "asc",
		"mode": "min"
	}
}
```

## 2.多值字段字符串排序

被分析器处理过的字符称为analyzed field ，analyzed字符串字符串字段同时也是多值字段。

为了使一个string字段可以进行排序，它必须只包含一个词：即完整的 not_analyzed 字符串(译者注：未经分析器分词并排序的原字符串)。 当然我们需要对字段进行全文本搜索的时候还必须使用被 analyzed 标记的字段 

同一个字段中同时包含这两种索引方式，只需要改变索引index的mapping，方法是在所有原有核心字段类型上，通过**通用参数field 对mapping进行修改**。

原有mapping

```JSON
"tweet": {
	"type": "string",
	"analyzer": "english"
}
```

改变后的多值字段mapping如下：

```JSON
"tweet": { //<1>
	"type": "string",
	"analyzer": "english",
	"fields": {
		"raw": { //<2>
			"type": "string",
			"index": "not_analyzed"
		}
	}
}
```

<1>tweet 字段用于全文本的analyzed索引方式不变

<2>新增的tweet.raw子字段索引方式是not_analyzed

给数据重建索引后，既可以使用tweet字段进行全文本搜索，也可以用tweet.raw字段进行排序。

```Java
GET /_search
{
	"query":{
    	"match":{
    		"tweet": "elasticsearch"	
    	}
	},
	"sort": "tweet.raw"
}
```

对analyzed字段进行强制排序会消耗大量内存

## 3.相关性

查询语句会为每个文档添加一个_score 字段，评分的计算方式取决于不同的查询类型，不同的查询语句用于不同的目的：fuzzy查询会计算与关键字的拼写相似程度，terms查询会计算找到的内容与关键词组成部分匹配的百分比。

Elasticsearch的相似度算法被定义为 TF/IDF，即检索词频率/反向文档频率，包括以下内容：

1. 检索词频率：

每个检索词在该字段出现的频率，频率越高，相关性越高。

2. 反向文档频率：

每个检索词在索引中出现的频率，频率越高，相关性越低，检验一个检索词在文档中的普遍重要性。

3. 字段长度准则：

字段的长度越长，相关性越低。

单个查询可以使用TF/IDF评分标准或其他方式，比如短语查询中检索词的距离或模糊查询里的检索词相似度 

如果多条查询子句被合并为一条复合查询语句，比如bool查询，每个查询子句计算得出的评分会被合并到总的相关性评分中。

### 理解评分标准

在每个查询语句中都有一个explain参数，将explain设为true就可以得到更为详细的信息。

explain 参数可以让返回结果添加一个 _score 评分的得来依据。 

词频率和文档频率实在每个分片中计算出来的，而不是每个索引中。

### Explain Api

当explain选项添加到某一文档上时，它会告诉你为何这个文档会被匹配，以及一个文档为何没有被匹配。

```JSON
GET /us/tweet/12/_explain
{
	"query" : {
		"filtered" : {
			"filter" : { "term" : { "user_id" : 2 }},
			"query" : { "match" : { "tweet" : "honeymoon" }}
		}
	}
}
```

## 4.字段数据

倒排索引在用于搜索时是非常卓越的，但却不是理想的排序结构。

当搜索的时候，我们需要用检索词去遍历所有文档。

当排序的时候，我们需要遍历文档中所有的值，需要做反倒序排列操作。

为了提高排序效率，Elasticsearch会将所有字段的值加载到内存中，这就叫做“数据字典”。

**ElasticSearch 将多有字段数据加载到内存中并不是匹配到那部分数据，而是索引下所有文档中的值，包括所有类型。**

将所有字段数据加载到内存是因为从硬盘反向倒排索引时非常缓慢的。

Elasticsearch中的字段数据常被应用到以下场景：

1. 对一个字段进行排序
2. 对一个字段进行聚合
3. 某些过滤，比如地理位置过滤
4. 某些与字段相关的脚本计算





