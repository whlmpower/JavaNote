# ElasticSearch权威指南（结构化查询章节）

## 1.请求体查询

同字符串查询一样，可以查询一个、多个或_all索引（indices）或类型（types）：

```Json
GET /index_2014*/type1, type2/_search
{}
```

使用post来替代GET

## 2.结构化查询Query DSL

Elasticsearch在一个简单的JSON接口中用结构化查询来展现Lucene绝大多数能力。使用结构化查询，需要传递query参数：

```JSON
GET /_search
{
    "query":YOUR_QUERY_HERE
}
```

空查询 {} 在功能上等同于使用match_all 查询子句，匹配所有文档

```JSON
GET /_search
{
    "query":{
        "match_all":{}
    }    
}
```

### 查询子句

一个查询子句一般使用这种结构

```JSON
{
    Query_name:{
        argument:value,
        argument:value,...
    }
}
```

或者指向一个指定的字段

```Json
{
    Query_name:{
        field_name:{
            argument:value,
            argument:value,...
        }
    }
}
```

查找tweet字段中找寻包含elasticsearch的成员

```JSON
GET /_search
{
    "query":{
        "match":{
            "tweet":"elasticsearch"
        }
    }
}
```

### 合并多子句

合并简单的子句为一个复杂的查询语句：

1. 叶子子句(leaf clauses)（比如match子句）用以在将查询字符串与一个字段(或多个字段)进行比较
2. 复合子句(compound)用以合并其他子句。例如，bool子句允许你合并其他的合法子句，must，must not，或者should

复合子句能合并任意其他查询子句，包括其他的复合子句，这就意味着复合子句可以互相嵌套，从而实现非常复杂的逻辑。

查询邮件正文中含有“business opportunity”的降表邮件或收件箱中政委还有“business opportunity”的非垃圾邮件。

```JSON
{
    "bool":{
        "must":{"match": {"email": "business opportunity" }},
        "should":[
            {"match": {"started":true}},
            { "bool": {
					"must": { "folder": "inbox" }},
					"must_not": { "spam": true }}
			}}
        ],
        "minimum_should_match": 1
    }
}
```



## 3.查询与过滤

两种结构化语句：结构化查询（Query DSL）和结构化过滤（Filter DSL）。使用目的不同而稍有差异。

过滤语句会询问每个文档的字段是否包含特定值。

查询语句会询问每个文档的字段值与特定值的匹配程度如何。

一条查询语句会计算每个文档与查询语句的相关性，会给出一个相关性评分 _score，并按照相关性对匹配到的文档进行排序。

### 性能差异

过滤语句得到的结果集--简单的文档列表，快速匹配运算并存入内存是十分方便。这些缓存的过滤结果集与后续请求的结合使用是非常高效的。

查询语句不仅要查找相匹配的文档，还要计算每个文档的相关性，查询语句比过滤语句更耗时，并且查询结果也是不可缓存的。

因**倒排索引**， 查询语句在百万级文档中的查询效率会与一条经过缓存的过滤语句旗鼓相当。

### 使用场景

原则上说，使用查询语句做全文本搜索或需要其他需要进行**相关性评分**的时候，剩下的全部用过滤语句。

### 3.重要的查询过滤语句

#### term 过滤

term主要用于精确匹配哪些值，比如数字、日期、布尔值或not_analyzed的字符串（未经分析的文本数据类型）

```JSON
{ "term": { "age": 26 }}
{ "term": { "date": "2014-09-01" }}
{ "term": { "public": true }}
{ "term": { "tag": "full_text" }}
```



#### terms 过滤

terms 允许指定多个匹配条件，某个字段指定了多个值，文档需要一起去做匹配

```JSON
{
	"terms": {
	"tag": [ "search", "full_text", "nosql" ]
	}
}
```



#### range过滤

按照指定范围查找一批数据

```JSON
{
    "range":{
        "age":{
            "gte":20,
            "lt":30
        }
    }
}
```

#### exists 和 missing 过滤

查找文档中是否包含指定字段或没有某个字段，类似于SQL is_null

```JSON
{
	"exists": {
		"field": "title"
	}
}
```

这两个过滤只是针对已经查询出一批数据来，但是想区分某个字段是否存在时使用。

#### bool 过滤

bool 过滤可以用来合并多个过滤条件查询结果的布尔逻辑，包含以下操作符：

must ： 多个查询条件的完全查询，相对于and

must_not：多个查询条件的相反匹配，not

should：至少有一个查询条件匹配， or

#### match_all 查询

查询所有文档，没有查询条件下的默认语句，所有的文档相关性都是相同的，得到的score 为1

#### match 查询

match查询是标准查询，全文本查询还是精确查询基本都用到它

match查询一个全文本字段，会在真正查询之前用分析器分析match一下查询字符：

```JSON
{
	"match": {
		"tweet": "About Search"
	}
}
```

如果用 match 下指定了一个确切值，在遇到数字，日期，布尔值或者 not_analyzed 的字符串时，它将为你搜索你给定的值： 

```JSON
{ "match": { "age": 26 }}
{ "match": { "date": "2014-09-01" }}
{ "match": { "public": true }}
{ "match": { "tag": "full_text" }}
```

只能指定某个确切字段某个确切值进行搜索。

#### multi_match 查询

查询允许你做match 查询的基础上同时搜索多个字段：

```Json
{
	"multi_match": {
		"query": "full text search",
		"fields": [ "title", "body" ]
	}
}
```

#### bool 查询

bool 查询与 bool 过滤相似，用于合并多个查询子句。bool过滤直接给出是否匹配成功，而bool查询计算每一个查询子句的_score（相关性分值）。

以下查询将会找到 title 字段中包含 "how to make millions"，并且 "tag" 字段没有被标为 spam 。 如果有标识为 "starred" 或者发布日期为2014年之前，那么这些匹配的文档将比同类网站等级高： 

```JSON
{
	"bool": {
		"must": { "match": { "title": "how to make millions" }},
		"must_not": { "match": { "tag": "spam" }},
		"should": [
			{ "match": { "tag": "starred" }},
			{ "range": { "date": { "gte": "2014-01-01" }}}
		]
	}
}
```

## 4.过滤查询

通常情况下，一条查询语句需要过滤语句的辅助，全文本搜索除外。

查询语句可以包括过滤子句。

#### 带过滤的查询

search API中只能包含query语句，所以我们需要用filtered来同时包含“query”和“filter”子句。

```JSON
GET /_search
{
    "query":{
        "filtered": {
			"query": { "match": { "email": "business opportunity" }},
			"filter": { "term": { "folder": "inbox" }}
		}
    }
}
```

#### 单条过滤语句

在query上下文中，如果只需要一条过滤语句，比如匹配全部邮件的时候，可以省略query子句

查询语句没有指定查询范围，默认使用match_all 查询。

### 查询语句中的过滤

在filter的上下文中使用一个query子句,下面的语句就是一条带有查询功能的过滤语句， 这条语句可以过滤掉看起来像垃圾邮件的文档 

```JSON
GET /_search
{
	"query": {
		"filtered": {
			"filter": {
				"bool": {
				"must": { "term": { "folder": "inbox" }},
				"must_not": {
					"query": { 
						"match": { "email": "urgent business proposal" }
					}
				}
			}
		}
	}
	}
}
```



只有在过滤中用到全文本匹配的时候才会使用这种结构。

## 6. 验证查询

validate API 可以验证一条查询语句是否合法

```JSON
GET /gb/tweet/_validate/query?explain
{
	"query": {
		"tweet" : {
			"match" : "really powerful"
		}
	}
}
```

explain 参数可以提供语句错误的更多详情 .





