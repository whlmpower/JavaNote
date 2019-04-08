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

使用flatMap方法的效果是，各个数组并不是分别映射成一个流，而是映射成流的内容。所有使用map(Arrays::Stream)时生成的单个流都被合并起来，即扁平化成一个流。

flatmap 方法会让你把一个流中的每个值都换成另一个流，然后把所有的流连接起来成为一个流。

#### 例子

给定[1,2,3] 和 [3,4] 返回[(1,3), (1, 4), (2, 3)...]

```Java
List<Integer> number1 = Arrays.asList(1, 2, 3);
List<Integer> number2 = Arrays.asList(3, 4);

List<int[]> pairs = 
    numbers1.stream()
    		.flatMap(i -> numbers2.stream() 
                    					.map(j -> new int[](i, j))) // 这一步返回的Stream<Integer[]>, 需要使用 flatMap扁平化处理
    		.collect(toList());
```

在前一步的基础上，返回总和能被3整除的数对

```Java
List<Integer> number1 = Arrays.asList(1, 2, 3);
List<Integer> number2 = Arrays.asList(3, 4);

List<int[]> pairs = 
    numbers1.stream()
    		.flatMap(i -> numbers2.stream()
                     					.filter(j -> (i + j) % 3 == 0)
                    					.map(j -> new int[](i, j))) 
    		.collect(toList());
```

### 查找和匹配

Stream 通过allMatch 、anyMatch 、noneMatch、findFirst、findAny 方法

anyMatch 是否有一个元素匹配

allMatch 是否都匹配

noneMatch 没有一个匹配

**短路特性**

findAny 返回任意一个

findFirst 返回第一个元素

**找到第一个元素在并行上限制很多，不关心返回元素是哪个，使用findAny，并行流时限制较少**

### 归约

#### 求和

reduce操作将这种重复应用模式做了抽象

```Java
int sum = numbers.stream().reduce(0, (a, b) -> a + b);
```

reduce接收两个参数：一个是初始值，另一个是BinaryOperator

Integer类中有一个静态的sum方法来对两个数求和

```Java
int sum = numbers.stream().reduce(0, Integer::sum);
```

reduce还有一个重载的变体，不接受初始值，但是会返回一个Optional对象

```java
Optional<Integer> sum = numbers.stream().reduce((a, b) -> (a + b));
```

#### 最大值最小值

```Java
Optional<Integer> max = numbers.stream().reduce(Integer::max);
```

**优势** 相比于迭代求和，使用reduce的好处在于，迭代被内部迭代抽象掉了，让内部实现得以选择并执行reduce。

迭代式求和要更新共享变量sum，不容易实现并行化。

```java
int sum = numbers.parallelStream().reduce(0, Integer::sum);
```

传递给reduce的Lambda不能更改状态，且操作需要满足结合率

**流操作：无状态和有状态**

诸如map或者filter等操作 是无状态的

诸如reduce、sum、max操作需要内部状态来累积结果

诸如sort或者distinct 从流中排序和删除重复项时都需要知道先前的历史，是有状态的

#### 实例

去掉distinct，改用toSet()，这样会把流转换为集合

返回所有交易员的姓名字符串，按照字母顺序排序

```Java
String traderStr = 
    	transactions.stream()
    				.map(transaction -> transaction.getTrade().getName())
    				.distinct()
    				.sorted()
    				.reduce("", (n1, n2) -> n1 + n2);
```

注意，此解决方法效率不高（字符串反复连接，每次迭代都需要建立一个新的String对象）

可以选择使用以下解决办法，内部使用了Stringbuilder

```Java
String traderStr = 
    	transactions.stream()
    				.map(transaction -> transaction.getTrade().getName())
    				.distinct()
    				.sorted()
    				.collect(joining());
```

### 数值流

使用Integer::sum 问题在于暗含的装箱成本，使用IntStream DoubleStream LongStream 避免了暗含的装箱成本。

```Java
int calories = menu.stream()
    			.mapToInt(Dish::getCalories)
    			.sum();
```

mapToInt 返回一个IntStream

将原始流转换为一般流（每个int都会装箱成一个Integer），可以使用boxed方法

```Java
IntStream intStream = menu.stream().mapToInt(Dish::getCalories);
Stream<Integer> stream = intStream.boxed();

```

OptionalInt 区分没有元素的流和最大值真的是0的流

```Java
OptionalInt maxCalories = menu.stream()
    							.mapToInt(Dish::getCalories)
    							.max();
```

range 和rangeClosed生成范围

```Java
IntStream evenNumbers = IntStream.rangeClosed(1, 100)
								.filter(n -> n % 2 == 0);
```

#### 勾股数

```Java
Stream<double[]> pythagoreanTriples2 = 
    IntStream.rangeClosed(1, 100).boxed()
    			.flatMap(a -> 
                        IntStream.rangeClosed(a, 100)
                        			.mapToObj(
                                    		b -> new Double[] {a, b, Math.sqrt(a *a + b * b )}).filter(t ->  t[2] % 1 == 0));
```

### 创建流

Stream.of 静态方法显示创建一个流

由数组创建流 Arrays.stream(numbers)

由文件生成流 Files.lines

```Java
long uniqueWords = 0;
try(Stream<String> lines = Files.lines(Paths.get("data.txt"), Charset.defaultCharset())){
    uniqueWords = lines.flatMap(line -> Arrays.stream(line.split(" ")))
        .distinct().count();
}catch(IOException){
    
}
```

你可以使用Files.lines得到一个流，其中的每个元素都是给定文件中的一行 可以对line调用split方法将行拆分成单词。应该注意的是，你该如何使用flatMap产生一个扁平的单词流，而不是给每一行生成一个单词流。最后，把distinct和count方法链接起来，数数流中有多少各不相同的单词。 

#### 由函数生成无限流

Stream.iterate 和 Stream.generate, 一般来说使用limit进行限制，避免无穷打印

**迭代**

```Java
Stream.iterate(0, n -> n + 2)
    		.limit(10).forEach(System.out::println);
```

iterate 接收一个初始值，和一个Lambda(unaryOperator<t> 类型)

```Java
Stream.iterate(new int[]{0, 1},
t -> new int[]{t[1],t[0] + t[1]})
.limit(10)
.map(t -> t[0])
.forEach(System.out::println);
```

**生成**

generate 接收一个Supplier<T> 类型的Lambda

```Java
IntSupplier fib = new IntSupplier(){
	private int previous = 0;
	private int current = 1;
	public int getAsInt(){
	int oldPrevious = this.previous;
	int nextValue = this.previous + this.current;
	this.previous = this.current;
	this.current = nextValue;
	return oldPrevious;
	}
};
IntStream.generate(fib).limit(10).forEach(System.out::println);
```

此段代码创建了一个IntSupplier的实例，此对象有可变的状态。它在两个实例变量中记录了前一个斐波纳契项和当前的斐波纳契项。 getAsInt在调用时会改变对象的状态，由此在每次调用时产生新的值。相比之下， 使用iterate的方法则是纯粹不变的：它没有修改现有状态，但在每次迭代时会创建新的元组。 

