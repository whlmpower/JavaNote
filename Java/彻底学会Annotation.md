
# Java 中的注解详解

## 什么是注解
注解：Annotation….

注解其实就是代码中的特殊标记，这些标记可以在编译、类加载、运行时被读取，并执行相对应的处理。  

## 为什么要用到注解
注解可以给类、方法上注入信息

## 基本的Annotation
java.lang 包下面存在5个基本的Annotation
### @Override
### @Deprecated 
### @SuppressWarnings
让编译器不给予我们警告，比如在使用集合的时候，如果没有指定泛型，那么会提示安全检查的警告

### @SafeVarargs 
堆污染警告，当把一个不是泛型的集合赋值给一个带泛型的集合时候，这种情况就很容易发生堆污染    
### @FunctionalInterface
@FunctionalInterface 用来指定该接口是函数式接口   

## 自定义注解基础
### 标记Annotation
没有任何成员变量的注解称为标记注解， @Override就是一个标记注解
```Java
public @interface MyAnnotation{

}
```
### 元数据Annotation
我们自定义的注解是可以带成员变量的，定义这种带成员变量的注解叫做元数据Annotation
```Java
public @interface MyAnnotation{
    String userName();
    int age();
}
```
注意： 在注解上定义成员变量的只能是String 数组 Class 枚举类 注解 

## 使用自定义注解
### 常规使用
下面我有一个add的方法，需要username和age参数，我们通过注解来让该方法拥有这两个变量！

```Java
@MyAnnotation(userName = "zhongfucheng", age=20)
public void add(String userName, int age){

}
```

### 默认值
在声明注解属性的时候，给出默认值
```Java
public @interface MyAnnotation{
    String userName() default "zicheng";
    int age() default 23;
}
```
在使用注解的时候就不需要给出具体的值了

```Java
@MyAnnotation
public void add(String userName, int age){
    
}
```
### 注解属性为Value
如果注解上只有一个属性，并且属性的名称为value，在使用的时候，可以不写value，直接进行赋值即可   
```Java
publice @interface MyAnnotation{

}

@MyAnnotation("zhongfucheng")
public void find(String id){

}
```

## 把自定义注解的属性基本信息注入到方法上
上面我们已经使用到了注解，但是目前为止注解上的信息和方法上的信息是没有任何关联的。我们自己写的自定义注解是需要我们自己来进行处理的，利用 **反射** 技术：

- 反射出该类的方法
- 通过方法得到注解上具体的信息
- 将注解上的信息注入到方法上  

```Java
//反射出该类的方法
Class aClass = Demo2.Class;
Method method = aClass.getMethod("add", String.class, int.class)

//通过该方法得到注解上的具体信息
MyAnnotation annotation = method.getAnnotation(MyAnnotation.class);
String userName = annotation.userName();
int age = annotation.age();

// 将注解上的信息注入到方法上
Object o = aClass.newInstance();
method.invoke(o, userName, age);
```
但是在执行的时候，会出现异常   
此时需要添加一个注解信息
```Java 
@Retention(RetentionPolicy.RUNTIME)
```

## JDK 中的元Annotation
在annotation包下面的好几个元Annotation都是用于修饰其他的Annotation定义的

### @Retention
@Retention 只能用于修饰其他的Annotation，用于指定被修饰的Annotation被保留多长时间  
@Retention 包含了一个RetentionPolicy 类型的value变量，所以在使用它的时候，必须要为value成员变量赋值
```Java
public enum RetentionPolicy {
    /**
     * Annotations are to be discarded by the compiler.
     */
    SOURCE,

    /**
     * Annotations are to be recorded in the class file by the compiler
     * but need not be retained by the VM at run time.  This is the default
     * behavior.
     */
    CLASS,

    /**
     * Annotations are to be recorded in the class file by the compiler and
     * retained by the VM at run time, so they may be read reflectively.
     *
     * @see java.lang.reflect.AnnotatedElement
     */
    RUNTIME
}
```
java文件有三个时期：编译,class,运行。@Retention默认是class

前面我们是使用反射来得到注解上的信息的，因为@Retention默认是class，而反射是在运行时期来获取信息的。因此就获取不到Annotation的信息了。于是，就得在自定义注解上修改它的RetentionPolicy值    

### @Target
@Target 也是只能用于修饰另外的Annotation，它用于指定被修饰的Annotation用于修饰哪些程序单元

@Target 是只有一个value成员变量的，该成员变量的值是以下的
```Java
public enum ElementType {
    /** Class, interface (including annotation type), or enum declaration */
    TYPE,

    /** Field declaration (includes enum constants) */
    FIELD,

    /** Method declaration */
    METHOD,

    /** Parameter declaration */
    PARAMETER,

    /** Constructor declaration */
    CONSTRUCTOR,

    /** Local variable declaration */
    LOCAL_VARIABLE,

    /** Annotation type declaration */
    ANNOTATION_TYPE,

    /** Package declaration */
    PACKAGE
}
```   
如果@Target指定的是ElementType.ANNOTATION_TYPE，那么该被修饰的Annotation只能修饰Annotaion   
### @Documented
@Documented 用于指定被该Annotation修饰的Annotation类将被Javadoc工具提取成文档    

### @Inherited 
@Inherited 也是用来修饰其他的Annotation的，被修饰过的Annotation将具有继承性    
例子：   
1. @xxx是我自定义的注解，我现在使用的@xxx注解在Base类上使用    
2. 使用@Inherited 修饰 @xxx注解   
3. 当有类继承了Base类的时候，该类自动拥有@xxx注解   
## 注入对象到方法或者成员变量上
### 把对象注入到方法上  
前面我们已经可以使用注解将基本信息注入到方法上了，现在我们要使用将对象注入到方法上   

#### 模拟场景
Person类，定义userName和age，拥有get 和  set方法
```Java
public class Person {

    private String username;
    private int age;

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }
}
```  
PersonDao类，PersonDao类定义了Person对象，拥有person的setter和getter方法    
```Java
public class PersonDao {

    private Person person;

    public Person getPerson() {
        return person;
    }

    public void setPerson(Person person) {
        this.person = person;
    }
}
```  
现在我要做的就是：使用注解将Person对象注入到setPerson()方法中，从而设置了PersonDao类的person属性    
```Java
public class PersonDao{
    private Person person;

    public Person getPerson(){
        return person;
    }

    @InjectPerson(userName = "zhongfucheng", age = 20)
    public void setPerson(Person person){
        this.person = person;
    }
}
```
*** 步骤： ***

1. 自定义一个注解，属性和JavaBean类一致
```Java
@Retention(RetentionPolicy.RUNTIME)
public @interface InjectPerson{
    String userName();
    int age();
}
```
2.  编写注入工具
```Java
// 1. 使用内省【后面需要得到属性的写方法】，得到想要注入的属性
PropertyDescriptor descriptor = new PropertyDescriptor("Person", PersonDao.class);

// 2. 得到想要注入的属性的具体对象
Person person = descriptor.getPropertyType().newInstance();

//3. 得到该属性的写方法【setPerson】
Method method = descriptor.getWriteMethod();

//4. 得到写方法上面的注解
Annotation annotation = method.getAnnotation(InjectPerson.class);

//5. 得到注解上面的信息【注解的成员变量就是用方法来定义的】
Method[] methods = annotation.getClass().getMethods();

// 6. 将注解上面的信息填充到person对象上
for(Method m : methods){

    // 得到注解上属性的名字【age 或者 name】
    String name = m.getName();

    // 看看Person对象有没有与之对象的方法【setAge setName】
    try{
            // 6.1 这里假设，有与之对应的写方法，得到写方法
            PropertyDescriptor descriptor1 = new PropertyDescriptor(name, Person.class);
            Method method1 = descriptor1.getWriteMethod();

            //得到注解中的值
            Object o = m.invoke(annotation, null);
            //调用Person 对象的 setter方法，将注解上的值设置进去
            method1.invoke(person, o);
        }catch(Exception e){
            // 6.2 Person 对象没有与之对应的方法，会跳到catch来，我们要让他继续遍历注解就好
            continue;
        }
    //程序遍历结束，person对象已经填充完数据了
    //7. 将Person对象赋值给PersonDao
    PersonDao personDao = new PersonDao();
    method.invoke(personDao, person);
    System.out.println(personDao.getPerson().getUsername());
    System.out.println(personDao.getPerson().getAge());
}

```
3. 总结
 - 得到想要注入的属性
 - 得到该属性的对象
 - 得到属性对应的写方法
 - 通过写方法得到注解
 - 获取注解详细的信息
 - 将注解的信息注入到对象上
 - 调用属性写方法，将已填充数据的对象注入到方法中
 
 作者：Java3c    
 [参考博客](https://mp.weixin.qq.com/s/n-P8W8OzcKIg3UiFC-JycA)








