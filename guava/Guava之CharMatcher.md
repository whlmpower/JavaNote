
## 字符串匹配器：CharMatcher     

|         方法        |                                   描述                                   |
|---------------------|--------------------------------------------------------------------------|
| anyOf(CharSequence) | 表明你想要匹配真的字段 CharMatcher.anyOf("aeiou") 可以匹配小写元音字母。 |
| is(Char)            | 表明你想匹配的一个确定的字符。                                           |
| inRange(char, char) | 表明你想匹配的一个字符范围，例如：CharMatcher.inRange('a', 'z')。        |
|                     |                                                                          |

此外，CharMatcher 还有这些方法：negate()、and(CharMatcher)、or(CharMatcher)。这些方法可以为 CharMatcher 提供方便的布尔运算。   

### 使用CharMatcher

CharMatcher 提供了很多方法来对匹配的字符序列 CharSequence 进行操作。以下只是列出了一些常用方法。

|                   方法                  |                                                             描述                                                            |
|-----------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| collapseFrom(CharSequence, char)        | 将一组连续匹配的字符串替换为一个指定的字符。例如：WHITESPACE.collapseFrom(string, ' ') 可以将连续的空字符串替换为单个字符。 |
| matchesAllOf(CharSequence)              | 测试字符序列是否全部匹配。例如：ASCII.matchesAllOf(string) 可以测试字符是否全部是 ASCII。                                   |
| removeFrom(CharSequence)                | 将匹配的字符序列移除                                                                                                        |
| retainFrom(CharSequence)                | 将没有匹配的字符序列移除                                                                                                    |
| trimFrom(CharSequence)                  | 去除开头和结尾匹配的部分                                                                                                    |
| replaceFrom(CharSequence, CharSequence) | 将匹配的字符替换为给定的序列                                                                                                |
|                                         |                                                                                                                             |



### 方法分类
根据函数的返回值和名称将这些方法分为三类。    
第一类是判定型函数，判断 CharMacher 和入参字符串的匹配关系。     
```Java
CharMatcher.is('a').matchesAllOf("aaa");//true
CharMatcher.is('a').matchesAnyOf("aba");//true
CharMatcher.is('a').matchesNoneOf("aba");//true
```  
第二类是计数型函数，查找入参字符串中第一次、最后一次出现目标字符的位置，或者目标字符出现的次数，比如 indexIn，lastIndexIn 和 countIn。    
```Java
CharMatcher.is('a').countIn("aaa"); // 3
CharMatcher.is('a').indexIn("java"); // 1
```     
第三类就是对匹配字符的操作。包括 removeFrom、retainFrom、replaceFrom、trimFrom、collapseFrom 等。       
```Java
CharMatcher.is('a').retainFrom("bazaar"); // "aaa"
CharMatcher.is('a').removeFrom("bazaar"); // "bzr"
CharMatcher.anyOf("ab").trimFrom("abacatbab"); // "cat"
```
### 源码分析
源码有很多行，主要的逻辑是这样的： 
* CharMatcher 是个抽象类，内部有一些私有的实现类，通过一些工厂函数 is、anyOf 等工厂函数创建对应的实例（固定的是单例，变化的会创建一个具体实例）。 
* 不同的实例主要实现的是 CharMatcher 中的 matches 方法，这样就实现了不同策略的匹配器。 
* 基于上述匹配方法 matches，可以进行统计工作（countIn等）、查找工作（indexIn等）、修改工作（trimFrom）等。