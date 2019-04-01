# Elasticsearch权威指南（映射和分析章节）

映射机制用于进行字段类型确认，将每个字段匹配为一种确定的数据类型（string，number, boolean, date等）

分析机制用于进行全文文本分词，以建立供搜索用的反向索引。

## 1.数据类型差异

我们的数据在_all 字段的索引方式和date字段的索引方式不同导致。

**查看mapping**  对gb索引中的tweet类型进行mapping

```JSON
GET /gb/_mapping/tweet
```

返回

```JSON
{
"gb": {
	"mappings": {
		"tweet": {
			"properties": {
				"date": {
					"type": "date",
					"format": "dateOptionalTime"
				},
				"name": {
					"type": "string"
				},
				"tweet": {
					"type": "string"
				},
				"user_id": {
					"type": "long"
				}
			}
		}
	}
}
}
```

Elasticsearch 对字段类型进行猜测，动态生成了字段和类型的映射关系。

date类型字段和String类型字段的索引方式不同，因此导致查询结果的不同。

**核心数据类型(string, numbersm booleans 和dates)** 以不同的方式进行索引

**确切值和全文文本两者的区别是区分搜索引擎和其他数据库的根本差异**

## 2.确切值和全文文本

确切值是确定的；全文文本是文本化的数据。

为了方便在全文文本字段中进行这些类型的查询，Elasticsearch首先对文本分析，然后使用结果建立一个倒排索引。

## 3.倒排索引

倒排索引由在文档中出现的唯一单词列表，以及对于每个单词在文档中的位置组成。

为了创建倒排索引，我们首先切分每个文档的content字段为单独的单词（terms 或者 tokens）

**我们使用相同的标准化规则处理字符串中content字段和文档中content字段**

## 4.分析和分析器

分析的过程：

1. 首先，标记化一个文本块为适用于倒排索引单独的词（term）
2. 然后标准化这些词为标准形式，提高他们的“可搜索性”或“查全率”

**字符过滤器**（character filter）：标记化前处理字符串，去除HTML标记

**分词器**（tokenizer）：标记化为独立的词，简单的分词器可以根据空格或逗号将单词分开

**标记过滤**（token filter）：修改次，转换大小写，去掉 a an the, 增加词（同义词）

### 内建分析器

**标准分析器  简单分析器   空格分析器   语言分析器   **   

### 当分析器被使用时

当我们索引一个文档，全文字段会被分析为单独的词来创建倒排索引。当我们在全文字段搜索时，我忙要让查询字符串经过同样的分析流程处理，确保这些词在索引中存在。

1. 查询全文字段，查询将使用相同的分析器来查询字符串，以产生正确的词列表
2. 查询确切值，查询将不分析查询字符串，但是可以进行指定

```JSON
GET /_search?q=2014
GET /_search?q=2014-09-15
```

当我们在 _all 字段中查询 2014-09-15 ，首先分析查询字符串，产生匹配任一词 2014 、 09 或 15 的查询语句，它依旧匹配12个推文，因为它们都包含词 2014 。 

```JSON
GET /_search?q=date:2014-09-15
```

当我们在 date 字段中查询 2014-09-15 ，它查询一个确切的日期，然后只找到一条推文 

### 测试分析器

使用analyze API 来查看文本是如何被分析的

```JSON
GET /_analyze?analyzer=standard&text=Text to analyze
```

返回

```JSON
{
	"tokens": [
		{
			"token": "text",
			"start_offset": 0,
			"end_offset": 4,
			"type": "<ALPHANUM>",
			"position": 1
		},
		{
			"token": "to",
			"start_offset": 5,
			"end_offset": 7,
			"type": "<ALPHANUM>",
			"position": 2
		},
		{
			"token": "analyze",
			"start_offset": 8,
			"end_offset": 15,
			"type": "<ALPHANUM>",
			"position": 3
		}
	]
}
```



token 是一个实际被存储在索引中的词。 position 指明词在原文本中是第几个出现的。 start_offset 和 end_offset 表示词在原文本中占据的位置。 

### 指定分析器

把字符串字段当作一个普通的字段——不做任何分析，只存储确切值，通过映射(mapping)人工设置这些字段

## 5.映射

索引中每个文档都有一个类型（type）。每个类型都拥有自己的映射（mapping）或者模式定义（schema definition）。一个映射定义了字段类型，每个字段的数据类型，以及字段被Elasticsearch处理的方式。映射还用于设置关联到类型上的元数据。

Elasticsearch将使用动态映射猜测字段类型，这些类型来自于JSON的基本数据类型

```JSON
GET /gb/_mapping/tweet
```

映射是Elasticsearch在创建索引时动态生成的

### 自定义字段映射

自定义一些特殊类型(Fields)，特别是字符串字段类型。自定义类型可以使你完成以下几点：

1. 区分全文字符串字段和准确字符串字段（分词与不分词）
2. 使用特定语言的分析器
3. 优化部分匹配字段
4. 指定自定义日期格式

string 类型的字段，默认的，考虑到包含全文本，它们的值在索引前要经过分析器分析，并且在全文搜索此字段前要把查询语句做分析处理。 对于String字段，最重要的映射参数是index和analyer

#### Index

**index** 参数控制字符串以何种方式呗索引，取值为以下三者中的一个：

**analyzed：**以全文形式索引此字段，首先分析这个字段，然后索引。

**not_analyzed：**索引这个字段，使之可以被搜索，但索引内容和指定值一样，不分析此字段

**no：**不索引这个字段。这个字段不能为搜索到。

**String类型字段的默认值是analyzed，如果想映射字段为确切值，设置它为not_analyzed**

```JSON
{
	"tag": {
		"type": "string",
		"index": "not_analyzed"
	}
}
```

其 他简单类型（long 、 double 、 date 等等） 也接受 index 参数，但相应的值只能 是 no 和 not_analyzed ，它们的值不能被分析。 

### analyer

对于analyzed类型的字符串字段，使用analyzer参数来指定哪一种分析器将在搜索和索引的时候使用。

### 更新映射

可以在第一次创建索引的时候指定映射的类型，也可以为新类型添加映射

**可以向已有映射中增加字段，但你不能修改它，新字段已经被合并至存在那个映射中**

## 6.复合核心字段类型

### 多值字段

索引一个标签数组来代替单一字符串

```JSON
{"tag";["search", "nosql"]}
```

数组中所有值必须为同一类型，创建一个新的字段，字段索引了一个数组，Elasticsearch将使用第一个值的类型来确定这个新字段的类型。

数组是做为多值字段被索引的，没有顺序。在搜索阶段你不能指定第一个或者最后一个数值，倒不如把数组当做一个值集合。

### 空字段

```JSON
"empty_string": "",
"null_value": null,
"empty_array": [],
"array_with_null_value": [ null ]
```

### 多层对象

```JSON
{
	"tweet": "Elasticsearch is very flexible",
	"user": {
		"id": "@johnsmith",
		"gender": "male",
		"age": 26,
		"name": {
			"full": "John Smith",
			"first": "John",
			"last": "Smith"
		}
	}
}
```



Elasticsearch 会动态的检测新对象的字段，并且映射他们为Object类型，将每个字段加到properties字段下

事实上， type 映射只是 object 映射 的一种特殊类型，我们将 object 称为根对象。它与其他对象一模一样，除非它有一些特殊 的顶层字段，比如 _source , _all 等等 

### 内部对象是怎样被索引的

一个Lucene文件包含一个键值对应的扁平表单

```JSON
{
	"tweet": [elasticsearch, flexible, very],
	"user.id": [@johnsmith],
	"user.gender": [male],
	"user.age": [26],
	"user.name.full": [john, smith],
	"user.name.first": [john],
	"user.name.last": [smith]
}
```

### 对象内部数组

```JSON
{
	"followers": [
		{ "age": 35, "name": "Mary White"},
		{ "age": 26, "name": "Alex Jones"},
		{ "age": 19, "name": "Lisa Smith"}
	]
}
```

此文件被偏平化为

```JSON
{
	"followers.age": [19, 26, 35],
	"followers.name": [alex, jones, lisa, smith, mary, white]
}
```

{age: 35} 与 {name: Mary White} 之间的关联会消失 

我们可以知道是否有26岁的followers,但是无法知道是否有26岁的追随者且名字为Alex Jones

**关联内部对象可以解决此问题**

