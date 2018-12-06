# Java中的十个“单行代码编程”    

1.对列表/数组中的每个元素都乘以2     
```Java
int[] ia = range(1,10).map(i -> i * 2).toArray();
List<Integer> result = range(1, 10).map(i  -> i * 2).boxed().collect(toList());
```
2.计算集合/数组中的数字之和
```Java
range(1, 1000).sum();
range(1, 1000).reduce(0, Integer::sum);
Stream.iterate(0, i -> i + 1).limit(1000).reduce(0, Integer::sum);
InStream.iterate(0, i -> i + 1).limit(1000).reduce(0, Integer::sum);
```    
关于reduce，参考链接[](https://blog.csdn.net/icarusliu/article/details/79504602)
3.验证字符串是否包含集合中的某一个字符串    
```Java
final List<String> keywords = Arrays.asList("brown", "fox", "dog", "pangram");
final String tweet = "The quick brown fox jumps over a lazy dog. #pangram http://www.rinkworks.com/words/pangrams.shtml";

keywords.stream().anyMatch(tweet::contains);
keywords.stream().reduce(false, (b, keyword) -> b || tweet.contains(keyword), (l, r) -> l || r);

```
**使用Guava 如何实现？**
```Java

```
4.读取文件内容   
```Java
try (BufferedReader reader = new BufferedReader(new FileReader("data.txt"))) {
  String fileText = reader.lines().reduce("", String::concat);
}   

try (BufferedReader reader = new BufferedReader(new FileReader("data.txt"))) {
  List<String> fileLines = reader.lines().collect(toCollection(LinkedList<String>::new));
} 

try (Stream<String> lines = Files.lines(new File("data.txt").toPath(), Charset.defaultCharset())) {
  List<String> fileLines = lines.collect(toCollection(LinkedList<String>::new));
}

```
5.输出歌曲《Happy Birthday to you!》 根据几个元素不同输出不同的字符串   
```Java
range(1, 5).boxed().map(i -> {out.print("Happy Birthday");  if (i == 3) return "dear name"; else return "To you";}).forEach(out::println);
```
6.过滤并分组集合中的数字
```Java
Map<String, List<Integer>> result = Stream.of(49, 58, 76, 82, 88, 90).collect(groupingBy(forPredicate(i -> i > 60, "passed", "failed")));
```  
7.获得并解析xml协议的WebService    
```Java
FeedType feed = JAXB.unmarshal(new URL("http://search.twitter.com/search.atom?&q=java8"), FeedType.class);
JAXB.marshal(feed, System.out);
```   
8.获得集合中最小/最大的数字
```Java
int min = Stream.of(14, 35, -7, 46, 98).reduce(Integer::min).get();
min = Stream.of(14, 35, -7, 46, 98).min(Integer::compare).get();
min = Stream.of(14, 35, -7, 46, 98).mapToInt(Integer::new).min();

int max = Stream.of(14, 35, -7, 46, 98).reduce(Integer::max).get();
max = Stream.of(14, 35, -7, 46, 98).max(Integer::compare).get();
max = Stream.of(14, 35, -7, 46, 98).mapToInt(Integer::new).max();
```
9.并行处理   
```Java
long result = dataList.parallelStream().mapToInt(line -> processItem(line)).sum();
```
10.集合上的各种查询   
```Java
List<Album> albums = Arrays.asList(unapologetic, tailgates, red);
// 筛选出至少有一个track评级4分以上的专辑，并按照名称排序后打印出来
albums.stream()
.filter(a -> a.tracks.stream().anyMatch(t -> (t.rating >= 4)))
.sorted(comparing(album -> album.name))
.forEach(album -> System.out.println(album.name));

//合并所有专辑的track
List<Track> allTracks = albums.stream()
.flatMap(album -> album.tracks.stream())
.collect(toList());

//根据track的评分对所有track分组
Map<Integer, List<Track>> tracksByRating = allTracks.stream()
 .collect(groupingBy(Track::getRating));

```

