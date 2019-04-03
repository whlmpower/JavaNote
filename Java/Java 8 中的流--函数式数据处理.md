# Java 8 中的流--函数式数据处理

## 1. 流的简介

流的定义：从支持数据处理操作的源生成的元素序列

集合讲的是数据，流讲的是计算

**流操作的两个重要特点：**

1. 流水线。很多流操作本身会返回一个流，这样多个操作就可以链接起来，形成一个大的流水线。一些优化：延迟和短路。流水线操作可以看作对数据源进行数据库式的查询。
2. 内部迭代。流的迭代操作是在背后进行的。

## 2.流与集合

集合与流之间的差异在于什么时候进行计算。集合中的元素都得先计算出来才能添加到集合中。流中的元素则是按需计算的。

### 只能遍历一次

和迭代器类似，流只能遍历一次，遍历完之后，流就已经被消费掉了。（假设它是集合之类可重复的源，如果是IO通道就没戏了）。

### 外部迭代与内部迭代

使用Collection接口需要用户去做迭代(for-each)，这种称之为外部迭代。

stream库的内部迭代可以自动选择一种适合你硬件的数据表示和并行实现；一旦通过写for-each而选择了外部迭代，基本就要自己管理所有的并行问题了。

## 3.流操作

### 中间操作

除非流水线上触发一个终端操作，否则中间操作不会执行任何处理。

### 终端操作

从流的流水线生产结果

### 使用流

流的使用包括：

1. 一个数据源来执行一个查询
2. 一个中间操作链，形成一条流水线
3. 一个终端操作，执行流水线，并能生产结果

## 4.使用流细节

Stream API让你快速完成复杂的数据查询，如筛选、切片、映射、查找、匹配、归约。

### 筛选和切片

Streams接口支持filter方法，接收返回boolean的函数作为参数。

distinct，根据流生成元素的hashCode和equals方法实现，返回元素各异的流。

limit(n)返回不超过给定长度的流。limit也可以用在无序流上，比如源是一个Set。

skip(n)返回扔掉了前n个元素的流。skip 和 limit是互补的。

### 映射

流支持map方法，接收一个函数作为参数，这个函数会被应用到每个元素上，并将其映射成一个新的元素。

#### 流的扁平化

给 定 单 词 列 表["Hello","World"]，你想要返回列表["H","e","l", "o","W","r","d"]。 

```Java
words.stream()
    .map(word -> word.split(""))
    .distinct()
    .collect(toList());
```

传递给map 方法的Lambda为每个单词返回一个String[] （String 列表），即返回了Stream<String[]>类型

使用Arrays.stream()

```Java
String[] arrayOfWords = {"Goodbye", "World"};
Stream<String> streamOfwords = Arrays.stream(arrayOfWords);

words.stream()
	.map(word -> word.split(""))// 将每个单词转换成由其字母构成的数组
	.map(Arrays::stream) // 每个数组都编程了一个独立的流，所以会存在两个流
	.distinct()
	.collect(toList());
```

使用flatMap

```Java
List<String> uniqueCharacters =
	words.stream()
		.map(w -> w.split(""))// 将每个单词转换成由其字母构成的数组
		.flatMap(Arrays::stream)//将各个生成流偏平化为单个流
		.distinct()
		.collect(Collectors.toList());
```

使用flatMap方法的效果是，各个数组并不是分别映射成一个流，而是映射成流的内容。扁平化成一个流。