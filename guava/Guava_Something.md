# 使用Google Guava 快乐编程    

## 以面向对象的思想处理字符串 Joiner/Splitter/CharMatcher    

String 提供的split方法，我们还得关心空字符串，还得考虑返回的结果中存在null 元素，只提供了前后的trim的方法（如果我想对中间元素进行trim）    

```Java
//连接器
private static final Joiner joiner = Joiner.on(",").skipNulls();

//分割器
private static final Splitter splitter = Splitter.on(",").trimResults().omitEmptyStrings();

public static void main(String[] args){
    String join = joiner.join(Lists.newArrayList("a", null, "b"));
    for(String tmp : splitter.split("a,   ,  b, ,")){
        System.out.println("|" + tmp + "|");
    }
}

```     
Joiner 是连接器，Splitter 是分割器，通过我们会把他们定义staticfinal，利用on生成对象后应用在String进行处理，可以进行复用的。Apache commons StringUtils 提供的都是static method。     
对于Joiner，常用的方法跳过NULL元素：skipNulls() 对于NULL元素使用其他替代： useForNull(String)       
对于Splitter，常用的方法是：trimResults/omitEmptyStrings()。 注意拆分的方式，有字符串，有正则，还有固定长度的分割        
