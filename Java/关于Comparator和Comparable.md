# 关于Comparator和Comparable

## Comparable 介绍
Comparable是在集合内部定义的方法实现的排序，位于java.lang下。

Comparable 接口仅仅只包括一个函数，它的定义如下：   

```Java
package java.lang;  
import java.util.*;  
  
  
public interface Comparable<T> {  
      
    public int compareTo(T o);  
}  
```

如果一个类实现了comparable接口，则意味着该类支持排序，如String、 Integer 自己就实现了Comparable接口，完成比较大小的操作。   
一个实现了Comparable接口的类或者对象，可以通过Collection.sort()或者Arrays.sort() 实现排序。    

## Comparator 介绍
Comparator是在集合外部实现的排序，位于java.util下。
Comparator接口包含了两个函数。 
```Java
package java.util;  
public interface Comparator<T> {  
  
    int compare(T o1, T o2);  
    boolean equals(Object obj);  
}  
```
如果我们需要控制某个类的次序，而该类本身不支持排序，那么我们可以新建一个该类的比较器来进行排序，这个比较器只需要实现Comparator即可    
Comparator 体现了一种策略模式(strategy design pattern)，就是不改变
对象自身，而是使用一种策略来改变它的行为。 
Comparator是外部比较器，Comparable是内部比较器    

## 使用实例
1. Comparable 使用示例
```Java
package test;  
  
import java.util.ArrayList;  
import java.util.Collections;  
import java.util.List;  
  
public class test {  
    public static void main(String[] args) {  
        List<UserInfo> list = new ArrayList<UserInfo>();  
        list.add(new UserInfo(1,21,"name1"));  
        list.add(new UserInfo(2,27,"name1"));  
        list.add(new UserInfo(3,15,"name1"));  
        list.add(new UserInfo(5,24,"name1"));  
        list.add(new UserInfo(4,24,"name1"));  
        //对该类排序  
        Collections.sort(list);  
        for(int i=0;i<list.size();i++){  
            System.out.println(list.get(i));  
        }  
    }  
}  
  
class UserInfo implements Comparable<UserInfo>{  
    private int userid;  
    private int age;  
    private String name;  
    public UserInfo(int userid, int age, String name) {  
        this.userid = userid;  
        this.age = age;  
        this.name = name;  
    }  
    public int getUserid() {  
        return userid;  
    }  
    public void setUserid(int userid) {  
        this.userid = userid;  
    }  
    public int getAge() {  
        return age;  
    }  
    public void setAge(int age) {  
        this.age = age;  
    }  
    public String getName() {  
        return name;  
    }  
    public void setName(String name) {  
        this.name = name;  
    }  
    @Override  
    public String toString(){  
        return this.userid+","+this.age+","+this.name;  
    }  
    @Override  
    public int compareTo(UserInfo o) {  
        //如果年龄相同，则比较userid，也可以直接  return this.age-o.age;  
        if(this.age-o.age==0){  
            return this.userid-o.userid;  
        }else{  
            return this.age-o.age;  
        }  
    }  
  
}  
```
2. Comparator 使用示例

```Java
package test;  
  
import java.util.ArrayList;  
import java.util.Collections;  
import java.util.Comparator;  
import java.util.List;  
  
public class test1 {  
    public static void main(String[] args) {  
        List<UserInfo> list = new ArrayList<UserInfo>();  
        list.add(new UserInfo(1,21,"name1"));  
        list.add(new UserInfo(2,27,"name2"));  
        list.add(new UserInfo(3,15,"name3"));  
        list.add(new UserInfo(5,24,"name4"));  
        list.add(new UserInfo(4,24,"name5"));  
        //new一个比较器  
        MyComparator comparator = new MyComparator();  
        //对list排序  
        Collections.sort(list,comparator);  
        for(int i=0;i<list.size();i++){  
            System.out.println(list.get(i));  
        }  
    }  
}  
class MyComparator implements Comparator<UserInfo>{  
    @Override  
    public int compare(UserInfo o1,UserInfo o2) {  
          
        if(o1.getAge()-o2.getAge()==0){  
            return o1.getUserid()-o2.getUserid();  
        }else{  
            return o1.getAge()-o2.getAge();  
        }  
    }  
}  
class UserInfo{  
    private int userid;  
    private int age;  
    private String name;  
    public UserInfo(int userid, int age, String name) {  
        this.userid = userid;  
        this.age = age;  
        this.name = name;  
    }  
    public int getUserid() {  
        return userid;  
    }  
    public void setUserid(int userid) {  
        this.userid = userid;  
    }  
    public int getAge() {  
        return age;  
    }  
    public void setAge(int age) {  
        this.age = age;  
    }  
    public String getName() {  
        return name;  
    }  
    public void setName(String name) {  
        this.name = name;  
    }  
    @Override  
    public String toString(){  
        return this.userid+","+this.age+","+this.name;  
    }  
}  
```