# Guava 之 Splitter   
## 概述
Java 中关于分词的工具类会有一些古怪的行为。例如：String.split 函数会悄悄地丢弃尾部分割符，而 StringTokenizer 处理5个空格字符串，结果将会什么都没有。

问题：",a,,b,".split(",") 的结果是什么？    
1. “”, “a”, “”, “b”, “”     
2. null, “a”, null, “b”, null      
3. “a”, null, “b”      
4. “a”, “b”     
5. 以上都不是     

正确答案是：5 以上都不是，应该是 "", "a", "",     "b"。只有尾随的空字符串被跳过。这样的结果很令人费解。    

Splitter 可以让你使用一种非常简单流畅的模式来控制这些令人困惑的行为。     
```
Splitter.on(',')
.trimResults()
.omitEmptyStrings()
.split("foo, bar, ,   qux");
```
以上代码将会返回Iterable<String>, 包含“foo”/"bar"/“qux”。 一个Splitter可以通过这些来进行划分： Pattern  char   String    CharMatcher    
如果你希望返回List,可以使用这样的代码Lists.newArrayList(splitter.split(string))## 
## 工厂函数

|                      方法                      |        描述        |
|------------------------------------------------|--------------------|
| Splitter.on(char)                              | 基于特定字符划分   |
| Splitter.on(CharMatcher)                       | 基于某些类别划分   |
| Splitter.on(String)                            | 基于字符串划分     |
| Splitter.on(Pattern)Splitter.onPattern(String) | 基于正则表达式划分 |
| Splitter.fixedLength(int)                      |    按指定长度划分，最后部分可以小于指定长度但不能为空                |

## 修改器 
|           方法           |                              描述                             |                                              例子                                             |
|--------------------------|---------------------------------------------------------------|-----------------------------------------------------------------------------------------------|
| omitEmptyStrings()       | 移去结果中的空字符串                                          | Splitter.on(',').omitEmptyStrings().split("a,,c,d") 返回 "a", "c", "d"                        |
| trimResults()            | 将结果中的空格删除，等价于trimResults(CharMatcher.WHITESPACE) | Splitter.on(',').trimResults().split("a, b, c, d") 返回 "a", "b", "c", "d"                    |
| trimResults(CharMatcher) | 移除匹配字符                                                  | Splitter.on(',').trimResults(CharMatcher.is('_')).split("_a ,_b_ ,c__") 返回 "a ", "b_ ", "c" |
| limit(int)               | 达到指定数目后停止字符串的划分                                | Splitter.on(',').limit(3).split("a,b,c,d") 返回 "a", "b", "c,d"                                                                                              |    
## Splitter.MapSplitter   
以下参考：  [Guava是个风火轮之基础工具(2)](http://www.importnew.com/15227.html)   
通过Splitter的withkeyValueSeparator 方法可以获得Joiner.Mapjoiner对象      
MapSplitter 只有一个公共方法，可以看到返回的对象是Map<String, String>
```Java
public Map<String, String> split(CharSequence sequence)
```
以下代码将返回这样的Map：{"1":"2", "3":"4"}。   
```Java
Splitter.on("#").withKeyValueSeparator(":").split("1:2#3:4");
```   
需要注意的是，MapSplitter对键值对格式有着严格的校验，下例会抛出java.lang.IllegalArgumentException 异常    
```Java
Splitter.on("#").withKeyValueSeparator(":").split("1:2#3:4:5");
```
因此，如果希望使用MapSplitter 来拆分KV结构的字符串，需要保证键值分隔符和键值对之间的分隔符不会称为键或值的一部分，也许是出于类似方面的考虑，MapSplitter 被加上了 @Beta 注解（未来不保证兼容，甚至可能会移除）。所以一般推荐使用 JSON 而不是 MapJoiner + MapSplitter。    

## 源码分析   
以下参考： [Guava 是个风火轮之基础工具(2)](http://www.importnew.com/15227.html)   
Splitter 的实现中有十分明显的策略模式和模板模式，有各种神乎其技的方法覆盖，还有 Guava 久负盛名的迭代技巧和惰性计算。    

### 成员变量
Splitter类有4个成员变量      
* CharMatcher trimmer：用于描述删除拆分结果的前后指定字符的策略。     
* boolean omitEmptyStrings：用于控制是否删除拆分结果中的空字符串。     
* Strategy strategy：用于帮助实现策略模式。      
* int limit：用于控制拆分的结果个数。     

### 策略模式 
Splitter可以根据字符、字符串、正则、长度还有Guava自己的字符匹配器CharMatcher来进行字符的拆分，基本上每种匹配模式的查找方式都不一样，但是字符拆分的基本框架又是不变的，所以策略模式正好合用。      

策略接口的定义很简单，就是传入一个Splitter和一个待拆分的字符串，返回一个迭代器。     
```Java
private interface Strategy{
    Iterator<String> iterator(Splitter splitter, CharSequence toSplit);
}
```   
每个工厂函数创建最后都会调用基本的私有构造函数，这个创建的过程中，主要是提供一个可以创建Iterator<String> 的 Strategy。   
```Java
private Splitter(Strategy strategy, boolean omitEmptyStrings, CharMatcher trimmer, int limit);
```
以 Splitter on(final CharMatcher separatorMatcher) 创建函数为例，这里返回的是 SplittingIterator (它是个抽象类，继承了 AbstractIterator，而 AbstractIterator 继承了 Iterator）。    
```Java
public static Splitter on(final CharMatcher separatorMatcher) {
    checkNotNull(separatorMatcher);
    return new Splitter(
        new Strategy() {
          @Override
          public SplittingIterator iterator(Splitter splitter, final CharSequence toSplit) {
            return new SplittingIterator(splitter, toSplit) {
              @Override
              int separatorStart(int start) {
                return separatorMatcher.indexIn(toSplit, start);
              }

              @Override
              int separatorEnd(int separatorPosition) {
                return separatorPosition + 1;
              }
            };
          }
        });
  }
```   
SplittingIterator 需要覆盖实现separatorStart 和  separatorEnd 两个方法才能实例化，这两个方法也是SplittingIterator用到的模板的重要组成。     

###  惰性迭代器与模板模式

（待补充）

[参考博客：](https://blog.csdn.net/u010738033/article/details/79594420)https://blog.csdn.net/u010738033/article/details/79594420