# 彻底学会Lambda
## Lambda 表达式的语法
基本语法：    
(parameters) -> expression;    
(parameters) -> {statements;}   

简单例子：
```Java
// 1. 不需要参数,返回值为 5  
() -> 5  
  
// 2. 接收一个参数(数字类型),返回其2倍的值  
x -> 2 * x  
  
// 3. 接受2个参数(数字),并返回他们的差值  
(x, y) -> x – y  
  
// 4. 接收2个int型整数,返回他们的和  
(int x, int y) -> x + y  
  
// 5. 接受一个 string 对象,并在控制台打印,不返回任何值(看起来像是返回void)  
(String s) -> System.out.print(s)  
```   
## 基本的Lambda例子
```Java
String[] atp = {"Rafael Nadal", "Novak Djokovic",  
       "Stanislas Wawrinka",  
       "David Ferrer","Roger Federer",  
       "Andy Murray","Tomas Berdych",  
       "Juan Martin Del Potro"};  
List<String> players =  Arrays.asList(atp);  
  
// 以前的循环方式  
for (String player : players) {  
     System.out.print(player + "; ");  
}  
  
// 使用 lambda 表达式以及函数操作(functional operation)  
players.forEach((player) -> System.out.print(player + "; "));  
   
// 在 Java 8 中使用双冒号操作符(double colon operator)  
players.forEach(System.out::println);  
```
匿名内部类可以使用Lambda表达式来代替   
```Java
// 使用匿名内部类  
btn.setOnAction(new EventHandler<ActionEvent>() {  
          @Override  
          public void handle(ActionEvent event) {  
              System.out.println("Hello World!");   
          }  
    });  
   
// 或者使用 lambda expression  
btn.setOnAction(event -> System.out.println("Hello World!"))
```
使用Lambda表达式来实现Runnable接口    
```Java
// 1.1使用匿名内部类  
new Thread(new Runnable() {  
    @Override  
    public void run() {  
        System.out.println("Hello world !");  
    }  
}).start();  
  
// 1.2使用 lambda expression  
new Thread(() -> System.out.println("Hello world !")).start();  
  
// 2.1使用匿名内部类  
Runnable race1 = new Runnable() {  
    @Override  
    public void run() {  
        System.out.println("Hello world !");  
    }  
};  
  
// 2.2使用 lambda expression  
Runnable race2 = () -> System.out.println("Hello world !");  
   
// 直接调用 run 方法(没开新线程哦!)  
race1.run();  
race2.run();  
```
## 使用Lambda表达式进行集合排序
不使用Lambda表达式的时候
```Java
String[] players = {"Rafael Nadal", "Novak Djokovic",   
    "Stanislas Wawrinka", "David Ferrer",  
    "Roger Federer", "Andy Murray",  
    "Tomas Berdych", "Juan Martin Del Potro",  
    "Richard Gasquet", "John Isner"};  
   
// 1.1 使用匿名内部类根据 name 排序 players  
Arrays.sort(players, new Comparator<String>() {  
    @Override  
    public int compare(String s1, String s2) {  
        return (s1.compareTo(s2));  
    }  
});  
```
使用Lambda表达式的时候   
```Java
// 1.2 使用 lambda expression 排序 players  
Comparator<String> sortByName = (s1, s2) -> (s1.compareTo(s2));
Array.sort(players, sortByName);

// 1.3 也可以采用如下形式:  
Arrays.sort(players, (String s1, String s2) -> (s1.compareTo(s2)));  
```
其他的排序  
```Java
// 1.1 使用匿名内部类根据 surname 排序 players  
Arrays.sort(players, new Comparator<String>() {  
    @Override  
    public int compare(String s1, String s2) {  
        return (s1.substring(s1.indexOf(" ")).compareTo(s2.substring(s2.indexOf(" "))));  
    }  
});  
  
// 1.2 使用 lambda expression 排序,根据 surname  
Comparator<String> sortBySurname = (String s1, String s2) ->   
    ( s1.substring(s1.indexOf(" ")).compareTo( s2.substring(s2.indexOf(" ")) ) );  
Arrays.sort(players, sortBySurname);  
  
// 1.3 或者这样,怀疑原作者是不是想错了,括号好多...  
Arrays.sort(players, (String s1, String s2) ->   
      ( s1.substring(s1.indexOf(" ")).compareTo( s2.substring(s2.indexOf(" ")) ) )   
    );  
  
// 2.1 使用匿名内部类根据 name lenght 排序 players  
Arrays.sort(players, new Comparator<String>() {  
    @Override  
    public int compare(String s1, String s2) {  
        return (s1.length() - s2.length());  
    }  
});  
  
// 2.2 使用 lambda expression 排序,根据 name lenght  
Comparator<String> sortByNameLenght = (String s1, String s2) -> (s1.length() - s2.length());  
Arrays.sort(players, sortByNameLenght);  
  
// 2.3 or this  
Arrays.sort(players, (String s1, String s2) -> (s1.length() - s2.length()));  
  
// 3.1 使用匿名内部类排序 players, 根据最后一个字母  
Arrays.sort(players, new Comparator<String>() {  
    @Override  
    public int compare(String s1, String s2) {  
        return (s1.charAt(s1.length() - 1) - s2.charAt(s2.length() - 1));  
    }  
});  
  
// 3.2 使用 lambda expression 排序,根据最后一个字母  
Comparator<String> sortByLastLetter =   
    (String s1, String s2) ->   
        (s1.charAt(s1.length() - 1) - s2.charAt(s2.length() - 1));  
Arrays.sort(players, sortByLastLetter);  
  
// 3.3 or this  
Arrays.sort(players, (String s1, String s2) -> (s1.charAt(s1.length() - 1) - s2.charAt(s2.length() - 1)));  
```

## 使用Lambda和Streams 
Stream是对集合的包装,通常和lambda一起使用。 使用lambdas可以支持许多操作,如 map, filter, limit, sorted, count, min, max, sum, collect 等等。 同样,Stream使用懒运算,他们并不会真正地读取所有数据,遇到像getFirst() 这样的方法就会结束链式语法。 在接下来的例子中,我们将探索lambdas和streams 能做什么。 我们创建了一个Person类并使用这个类来添加一些数据到list中,将用于进一步流操作。 Person 只是一个简单的POJO类:   
```Java
public class Person {  
  
private String firstName, lastName, job, gender;  
private int salary, age;  
  
public Person(String firstName, String lastName, String job,  
                String gender, int age, int salary)       {  
          this.firstName = firstName;  
          this.lastName = lastName;  
          this.gender = gender;  
          this.age = age;  
          this.job = job;  
          this.salary = salary;  
}  
// Getter and Setter   
// . . . . .  
}  
```
接下来,我们将创建两个list,都用来存放Person对象:  
```Java
List<Person> javaProgrammers = new ArrayList<Person>() {  
  {  
    add(new Person("Elsdon", "Jaycob", "Java programmer", "male", 43, 2000));  
    add(new Person("Tamsen", "Brittany", "Java programmer", "female", 23, 1500));  
    add(new Person("Floyd", "Donny", "Java programmer", "male", 33, 1800));  
    add(new Person("Sindy", "Jonie", "Java programmer", "female", 32, 1600));  
    add(new Person("Vere", "Hervey", "Java programmer", "male", 22, 1200));  
    add(new Person("Maude", "Jaimie", "Java programmer", "female", 27, 1900));  
    add(new Person("Shawn", "Randall", "Java programmer", "male", 30, 2300));  
    add(new Person("Jayden", "Corrina", "Java programmer", "female", 35, 1700));  
    add(new Person("Palmer", "Dene", "Java programmer", "male", 33, 2000));  
    add(new Person("Addison", "Pam", "Java programmer", "female", 34, 1300));  
  }  
};  
  
List<Person> phpProgrammers = new ArrayList<Person>() {  
  {  
    add(new Person("Jarrod", "Pace", "PHP programmer", "male", 34, 1550));  
    add(new Person("Clarette", "Cicely", "PHP programmer", "female", 23, 1200));  
    add(new Person("Victor", "Channing", "PHP programmer", "male", 32, 1600));  
    add(new Person("Tori", "Sheryl", "PHP programmer", "female", 21, 1000));  
    add(new Person("Osborne", "Shad", "PHP programmer", "male", 32, 1100));  
    add(new Person("Rosalind", "Layla", "PHP programmer", "female", 25, 1300));  
    add(new Person("Fraser", "Hewie", "PHP programmer", "male", 36, 1100));  
    add(new Person("Quinn", "Tamara", "PHP programmer", "female", 21, 1000));  
    add(new Person("Alvin", "Lance", "PHP programmer", "male", 38, 1600));  
    add(new Person("Evonne", "Shari", "PHP programmer", "female", 40, 1800));  
  }  
};  
```
现在我们使用forEach方法来迭代输出上述列表:  
```Java
System.out.println("所有程序员的姓名:");  
javaProgrammers.forEach((p) -> System.out.printf("%s %s; ", p.getFirstName(), p.getLastName()));  
phpProgrammers.forEach((p) -> System.out.printf("%s %s; ", p.getFirstName(), p.getLastName()));  
```
我们同样使用forEach方法,增加程序员的工资5%: 
```Java
System.out.println("给程序员加薪 5% :");  
Consumer<Person> giveRaise = e -> e.setSalary(e.getSalary() / 100 * 5 + e.getSalary());  
  
javaProgrammers.forEach(giveRaise);  
phpProgrammers.forEach(giveRaise);  
```
另一个有用的方法是过滤器filter() ,让我们显示月薪超过1400美元的PHP程序员:  
```Java
System.out.println("下面是月薪超过 $1,400 的PHP程序员:")  
phpProgrammers.stream()  
          .filter((p) -> (p.getSalary() > 1400))  
          .forEach((p) -> System.out.printf("%s %s; ", p.getFirstName(), p.getLastName()));  
```
我们也可以定义过滤器,然后重用它们来执行其他操作:
```Java
// 定义 filters  
Predicate<Person> ageFilter = (p) -> (p.getAge() > 25);  
Predicate<Person> salaryFilter = (p) -> (p.getSalary() > 1400);  
Predicate<Person> genderFilter = (p) -> ("female".equals(p.getGender()));  
  
System.out.println("下面是年龄大于 24岁且月薪在$1,400以上的女PHP程序员:");  
phpProgrammers.stream()  
          .filter(ageFilter)  
          .filter(salaryFilter)  
          .filter(genderFilter)  
          .forEach((p) -> System.out.printf("%s %s; ", p.getFirstName(), p.getLastName()));  
  
// 重用filters  
System.out.println("年龄大于 24岁的女性 Java programmers:");  
javaProgrammers.stream()  
          .filter(ageFilter)  
          .filter(genderFilter)  
          .forEach((p) -> System.out.printf("%s %s; ", p.getFirstName(), p.getLastName()));  
```
使用limit方法,可以限制结果集的个数:   
```Java
System.out.println("最前面的3个 Java programmers:");  
javaProgrammers.stream()  
          .limit(3)  
          .forEach((p) -> System.out.printf("%s %s; ", p.getFirstName(), p.getLastName()));  
  
  
System.out.println("最前面的3个女性 Java programmers:");  
javaProgrammers.stream()  
          .filter(genderFilter)  
          .limit(3)  
          .forEach((p) -> System.out.printf("%s %s; ", p.getFirstName(), p.getLastName()));  
```  
排序呢? 我们在stream中能处理吗? 答案是肯定的。 在下面的例子中,我们将根据名字和薪水排序Java程序员,放到一个list中,然后显示列表:
```Java
System.out.println("根据 name 排序,并显示前5个 Java programmers:");  
List<Person> sortedJavaProgrammers = javaProgrammers  
          .stream()  
          .sorted((p, p2) -> (p.getFirstName().compareTo(p2.getFirstName())))  
          .limit(5)  
          .collect(toList());  
  
sortedJavaProgrammers.forEach((p) -> System.out.printf("%s %s; %n", p.getFirstName(), p.getLastName()));  
   
System.out.println("根据 salary 排序 Java programmers:");  
sortedJavaProgrammers = javaProgrammers  
          .stream()  
          .sorted( (p, p2) -> (p.getSalary() - p2.getSalary()) )  
          .collect( toList() );  
  
sortedJavaProgrammers.forEach((p) -> System.out.printf("%s %s; %n", p.getFirstName(), p.getLastName()));  
```
如果我们只对最低和最高的薪水感兴趣,比排序后选择第一个/最后一个 更快的是min和max方法:
```Java
System.out.println("工资最低的 Java programmer:");  
Person pers = javaProgrammers  
          .stream()  
          .min((p1, p2) -> (p1.getSalary() - p2.getSalary()))  
          .get()  
  
System.out.printf("Name: %s %s; Salary: $%,d.", pers.getFirstName(), pers.getLastName(), pers.getSalary())  
  
System.out.println("工资最高的 Java programmer:");  
Person person = javaProgrammers  
          .stream()  
          .max((p, p2) -> (p.getSalary() - p2.getSalary()))  
          .get()  
  
System.out.printf("Name: %s %s; Salary: $%,d.", person.getFirstName(), person.getLastName(), person.getSalary())  
```
上面的例子中我们已经看到 collect 方法是如何工作的。 结合 map 方法,我们可以使用 collect 方法来将我们的结果集放到一个字符串,一个 Set 或一个TreeSet中:  
```Java
System.out.println("将 PHP programmers 的 first name 拼接成字符串:");  
String phpDevelopers = phpProgrammers  
          .stream()  
          .map(Person::getFirstName)  
          .collect(joining(" ; ")); // 在进一步的操作中可以作为标记(token)     
  
System.out.println("将 Java programmers 的 first name 存放到 Set:");  
Set<String> javaDevFirstName = javaProgrammers  
          .stream()  
          .map(Person::getFirstName)  
          .collect(toSet());  
  
System.out.println("将 Java programmers 的 first name 存放到 TreeSet:");  
TreeSet<String> javaDevLastName = javaProgrammers  
          .stream()  
          .map(Person::getLastName)  
          .collect(toCollection(TreeSet::new));  
```
Streams 还可以是并行的(parallel)。 示例如下:
```Java
System.out.println("计算付给 Java programmers 的所有money:");  
int totalSalary = javaProgrammers  
          .parallelStream()  
          .mapToInt(p -> p.getSalary())  
          .sum();  
```
我们可以使用summaryStatistics方法获得stream 中元素的各种汇总数据。 接下来,我们可以访问这些方法,比如getMax, getMin, getSum或getAverage:
```java
//计算 count, min, max, sum, and average for numbers  
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);  
IntSummaryStatistics stats = numbers  
          .stream()  
          .mapToInt((x) -> x)  
          .summaryStatistics();  
  
System.out.println("List中最大的数字 : " + stats.getMax());  
System.out.println("List中最小的数字 : " + stats.getMin());  
System.out.println("所有数字的总和   : " + stats.getSum());  
System.out.println("所有数字的平均值 : " + stats.getAverage());   
```
## 关于Predicate 和 Consumer 
### Predicate 
Predicate功能判断输入的对象是否符合某个条件。官方文档解释到：Determines if the input object matches some criteria.

了解Predicate接口作用后，在学习Predicate函数编程前，先看一下Java 8关于Predicate的源码：   
```Java
@FunctionalInterface
public interface Predicate<T> {

    /**
     * Evaluates this predicate on the given argument.
     *
     * @param t the input argument
     * @return {@code true} if the input argument matches the predicate,
     * otherwise {@code false}
     */
    boolean test(T t);

    default Predicate<T> and(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) && other.test(t);
    }

    default Predicate<T> negate() {
        return (t) -> !test(t);
    }

    default Predicate<T> or(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) || other.test(t);
    }


    static <T> Predicate<T> isEqual(Object targetRef) {
        return (null == targetRef)
                ? Objects::isNull
                : object -> targetRef.equals(object);
    }
}
```
从上面代码可以发现，Java 8新增了接口的默认(default)方法和(static)静态方法。在Java 8以前，接口里的方法要求全部是抽象方法。但是静态(static)方法只能通过接口名调用，不可以通过实现类的类名或者实现类的对象调用；默认(default)方法只能通过接口实现类的对象来调用。   
接下来主要来使用接口方法test，可以使用匿名内部类提供test()方法的实现，也可以使用lambda表达式实现test()。 
体验一下Predicate的函数式编程，使用lambda实现。其测试代码如下：  
```java
@Test
public void testPredicate(){
    java.util.function.Predicate<Integer> boolValue = x -> x > 5;
    System.out.println(boolValue.test(1));//false
    System.out.println(boolValue.test(6));//true
}
```
### Consumer

Consumer接口的文档声明如下：

An operation which accepts a single input argument and returns no result. Unlike most other functional interfaces, Consumer is expected to operate via side-effects.

即接口表示一个接受单个输入参数并且没有返回值的操作。不像其它函数式接口，Consumer接口期望执行带有副作用的操作(Consumer的操作可能会更改输入参数的内部状态)。

同样，在了解Consumer函数编程前，看一下Consumer源代码，其源代码如下：  

```java
@FunctionalInterface
public interface Consumer<T> {

    /**
     * Performs this operation on the given argument.
     *
     * @param t the input argument
     */
    void accept(T t);

    /**
     * Returns a composed {@code Consumer} that performs, in sequence, this
     * operation followed by the {@code after} operation. If performing either
     * operation throws an exception, it is relayed to the caller of the
     * composed operation.  If performing this operation throws an exception,
     * the {@code after} operation will not be performed.
     *
     * @param after the operation to perform after this operation
     * @return a composed {@code Consumer} that performs in sequence this
     * operation followed by the {@code after} operation
     * @throws NullPointerException if {@code after} is null
     */
    default Consumer<T> andThen(Consumer<? super T> after) {
        Objects.requireNonNull(after);
        return (T t) -> { accept(t); after.accept(t); };
    }
}
```
从上面代码可以看出，Consumer使用了Java 8接口新特性——接口默认(default)方法。接下来使用接口方法accept，体验一下Consumer函数编程。其测试代码如下：

```java
@Test
public void testConsumer(){
    User user = new User("zm");
    //接受一个参数
    Consumer<User> userConsumer = User1 -> User1.setName("zmChange");
    userConsumer.accept(user);
    System.out.println(user.getName());//zmChange
}
```

在Java 8之前的实现如下：

```java
@Test
public void test(){
    User user = new User("zm");
    this.change(user);
    System.out.println(user.getName());//输出zmChange
}

private void change(User user){
    user.setName("zmChange");
}
```

### Predicate 和  Consumer 综合运用
为了详细说明Predicate和Consumer接口，通过一个学生例子：Student类包含姓名、分数以及待付费用，每个学生可根据分数获得不同程度的费用折扣。

Student类源代码：  
```java
public class Student {

    String firstName;

    String lastName;

    Double grade;

    Double feeDiscount = 0.0;

    Double baseFee = 2000.0;

    public Student(String firstName, String lastName, Double grade) {
        this.firstName = firstName;
        this.lastName = lastName;
        this.grade = grade;
    }

    public void printFee(){
        Double newFee = baseFee - ((baseFee * feeDiscount)/100);
        System.out.println("The fee after discount: " + newFee);
    }
}
```
然后分别声明一个接受Student对象的Predicate接口以及Consumer接口的实现类。本例子使用Predicate接口实现类的test()方法判断输入的Student对象是否拥有费用打折的资格，然后使用Consumer接口的实现类更新输入的Student对象的折扣。  

```java
public class PredicateConsumerDemo {

    public static Student updateStudentFee(Student student, Predicate<Student> predicate, Consumer<Student> consumer){
        if (predicate.test(student)){
            consumer.accept(student);
        }
        return student;
    }

}
```
Predicate和Consumer接口的test()和accept()方法都接受一个泛型参数。不同的是test()方法进行某些逻辑判断并返回一个boolean值，而accept()接受并改变某个对象的内部值。updateStudentFee方法的调用如下所示：   
```java
public class Test {
    public static void main(String[] args) {
        Student student1 = new Student("Ashok","Kumar", 9.5);

        student1 = updateStudentFee(student1,
                //Lambda expression for Predicate interface
                student -> student.grade > 8.5,
                //Lambda expression for Consumer inerface
                student -> student.feeDiscount = 30.0);
        student1.printFee(); //The fee after discount: 1400.0

        Student student2 = new Student("Rajat","Verma", 8.0);
        student2 = updateStudentFee(student2,
                //Lambda expression for Predicate interface
                student -> student.grade >= 8,
                //Lambda expression for Consumer inerface
                student -> student.feeDiscount = 20.0);
        student2.printFee();//The fee after discount: 1600.0

    }
}
```
[参考博客](https://blog.csdn.net/renfufei/article/details/24600507)
[原文地址](https://www.developer.com/java/start-using-java-lambda-expressions.html)





