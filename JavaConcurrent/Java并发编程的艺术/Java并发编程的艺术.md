# Java并发编程的艺术

## 第一章 并发编程的挑战

### 1. 减少上下文切换

减少上下文的切换方法有无锁并发编程、CAS算法、使用最少线程和使用协程

无锁并发编程：将数据ID按照Hash算法取模分段，不同线程处理不同段的数据



### 2.避免死锁的常见方法

避免在一个线程同时获取多个锁

避免一个线程在锁内同时占用多个资源，尽量保证每个锁只占用一个资源

尝试使用定时锁，使用lock.tryLock(TimeOut)来替代内部锁机制

对于数据库锁，加锁和解锁必须在一个数据库连接里面，否则会出现解锁失败的情况

## 第二章 Java并发机制底层实现原理

### 1. Volatile 

**可见性**：当一个线程修改一个共享变量时，另一个线程能够读到这个修改的值

**Volatile的两个原则**：（1） Lock 前缀指令会引起处理器缓存回写到内存。（2）一个处理器的缓存回写到内存会导致其他处理器的缓存无效。

**追加字节优化性能**：如果队列的头节点和尾节点都不足64字节，处理器会将他们都读到同一个高速缓存行中，在多处理器下每个处理器都会缓存同样的头尾节点，当一个处理器视图修改头节点时，会将整个缓存行锁定，那么会在缓存一致性的机制作用下，会导致其他处理器不能访问自己高速缓存中的尾节点，而队列的入队和出队操作会不停地修改头尾节点，会在多处理器的情况下，影响到队列的入队和出队效率。

#### 对象头

![MarkWord 对象头](.\images\1553503009632.png)

### 偏向锁

**获取：**在对象头和栈帧中的锁记录中存储锁偏向的线程ID。不需要CAS加锁和解锁，简单测试对象头中的Mark Word里面是否存储指向当前线程的偏向锁。成功，获得锁；失败，测试是否是偏向锁：没有设置为1，采用CAS竞争；设置了尝试CAS将对象头的偏向锁指向当前线程。

**撤销：** 竞争出现才释放锁，撤销需要等待全局安全点。暂定拥有偏向锁的线程，检查持有偏向锁的线程是否存活，不存活，将对象头设置为无锁；存活，拥有偏向锁的栈将被执行，遍历偏向对象的锁记录，栈中的锁记录和对象头的Mark Word要么重新偏向于其他线程，要么恢复到无锁或者标记对象不适合作为偏向锁，最后唤醒暂停的线程 。

#### 轻量级锁

**获取：**栈帧中创建用于记录锁记录的空间，将对象头中的Mark Word复制到锁记录中。线程尝试使用CAS将对象头中Mark Word替换为指向锁记录的指针。成功，当前线程获取到锁；失败，当前线程自旋获取锁。

**释放：** CAS将锁记录替换回对象头，失败，膨胀为重量级锁。

## 第三章 Java内存模型

线程之间的通信机制有两种：共享内存和消息传递

为了保证内存可见性，Java编译器在生成指令序列的合适位置中会插入内存屏障指令来禁止重排序。

Happens-before 仅仅要求前一个操作对后一个操作可见，且前一个操作按顺序排在第二个操作之前。

**数据依赖性：**编译器和处理器不会改变存在数据依赖关系的两个操作的执行顺序 。

**as-if-serial:**不管怎么重排序，程序的执行结果不能被改变。

**顺序一致性：** 如果程序是正确同步的，程序的执行将具有顺序一致性。

JMM不保证对64位的long型和double型变量的写操作具有原子性。

### Volatile的内存语义

一个volatile变量的读，总是能看到(任意线程)对这个volatile变量最后的写入。

**volatile变量特性：**

可见性：对于一个volatile变量的读，总是能看到（任意线程）对这个volatile变量最后的写入。

原子性：对任意单个volatile变量的读/写具有原子性，但类似于volatile++这种复合操作不具备原子性。

**A线程写一个volatile变量后，B线程读同一个volatile变量。A线程在写volatile变量之前所有可见的共享变量，在B线程读同一个volatile变量后，将立即变得对B线程可见。**

#### Volatile 内存语义实现

![1553517845777](.\images\1553517845777.png)

当第二个操作是volatile写时，不管第一个操作是什么，都不能重排序。这个规则确保volatile写之前的操作不能被编译器重排序到volatile写之后。

当第一个操作是volatile读时，不管第二个操作是什么，都不能重排序。这个规则确保volatile读之后的操作不会被编译器重排序到volatile读之前。

当第一个操作是volatile写，第二个操作是volatile读时，不能重排序。

StoreStore屏障可以保证在volatile写之前，其前面的所有普通写操作已经对任意处理器可见了。这是因为StoreStore屏障将保障上面所有的普通写在volatile写之前刷新到主内存。

StoreLoad屏障避免volatile写与后面可能有的volatile读/写操作重排序。

 ### 锁的内存语义

线程A在释放锁之前所有可见的共享变量，在线程B获取到同一个锁之后，将立即变得对B线程可见。

**volatile 变量是ReentrantLock内存语义实现的关键。**

**公平锁和非公平锁释放时，最后都要写一个volatile变量的state；公平锁获取时，首先会去读volatile变量**。

concurrent包源代码实现，会发现一个通用化的实现模式：

首先，声明共享变量为volatile；

使用CAS的原子条件更新来实现线程之间的同步；

配合volatile的读/写和CAS所具有的volatile读和写的内存语义来实现线程之间的通信。

### Final域内存语义

final域重排序规则：

在构造函数内对一个final域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。 

初次读一个包含final域的对象的引用，与随后初次读这个final域，这两个操作之间不能重排序。 

写final域重排序规则可以确保：在对象引用为任意线程可见之前，对象的final域已经被正确初始化了。而普通域不具有这个保障。

读final域的重排序规则可以确保：在读一个对象final域之前，一定会先读包含这个final域的对象的引用。

**final域为一个引用类型** ：在构造函数内对一个final引用的对象成员的写入，与随后在构造函数外把这个被构造对象引用赋值给一个引用变量，这两个操作之间不能重排序。

**this逸出：** 在构造函数返回前，被构造对象的引用不能为其他线程所见，因为此时的final域可以还没有被初始化。

**增强final语义：** 只要对象是正确构造的(被构造的对象的引用在构造函数中没有逸出)，那么就不需要使用同步(指lock和volatile的使用)就可以保证任意线程都能看到这个final域在构造函数中被初始化之后的值。

### happens-before

**Join** 线程A执行操作ThreadB.join()并成功返回后，线程B中任意操作都将对线程A可见。

### 双重检查锁定与延迟初始化

```java
public class DoubleCheckedLocking{
    private static Instance instance;
    public static Instance getInstance(){
        if(instance == null){
            synchronized(DoubleCheckedLocking.class){
                if(instance == null){
                    instance = new Instance();
                }
            }
        }
        return instance;
    }
}
```

问题在于第4行，代码读取到instance不为null，instance引用对象可能还没有完成初始化。

```Java
instance = new Instance();// 创建对象包含以下几步

memory = allocate();//1. 分配对象的内存空间
ctorInstance(memory);//2. 初始化对象
instance = memory; // 3. 设置instance指向刚分配的内存地址
```

2 和 3之间可能存在重排序，可能的执行时序如下：

![1553592650806](.\images\%5CUsers%5CAdministrator%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5C1553592650806.png) 

线程B将会访问到一个还未初始化的对象

**基于volatile的解决方案**

```java
public class SafeDoubleCheckedLocking{
    private volatile static Instance instance;
    public static Instance getInstance(){
        if(instance == null){
            synchronized(DoubleCheckedLocking.class){
                if(instance == null){
                    instance = new Instance();
                }
            }
        }
        return instance;
    }
}
```

当声明对象的引用为volatile后，第3行伪代码中的2和3之间的重排序在多线程环境中将会被禁止。

其原理在于：普通写和volatile写之间之间不能够重排序。

**基于类初始化的解决方案**

JVM在类初始化阶段，会获取一个锁，这个锁可以同步多个线程对同一个类的初始化。

```Java
public class InstanceFactory{
    private static class InstanceHolder{
        public static Instance instance = new Instance();
    }
    public static Instance getInstance(){
        return InstanceHolder.instance;//这里将导致InstanceHolder类被初始化
    }
}
```

这种方案是允许伪代码中2 和 3 重排序，但不允许非构造线程看到这个重排序。

基于volatile的双重检查锁定的方案还有一个格外的优势：除了可以对静态字段实现延迟初始化外，还可以对实例字段实现延迟初始化。

## 第四章 Java并发编程基础

### 启动或终止线程

针对频繁阻塞（休眠或者I/O操作）的线程需要设置较高优先级，而偏重计算（需要较多CPU时间或者偏运算）的线程则设置较低的优先级，确保处理器不会被独占。 

**线程状态**

![1553594184777](.\images\%5CUsers%5CAdministrator%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5C1553594184777.png)

**Java 线程变迁**

![1553594357170](.\images\%5CUsers%5CAdministrator%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5C1553594357170.png)

**阻塞状态**是线程阻塞在进入synchronized关键字修饰的方法或代码块（获取锁）时的状态，但是阻塞在java.concurrent包中Lock接口的线程状态却是等待状态，因为java.concurrent包中Lock接口对于阻塞的实现均使用了LockSupport类中的相关方法 。

Daemon线程被用作完成支持性工作，但是在Java虚拟机退出时Daemon线程中的finally块并不一定会执行，**不能依靠Damend线程的finally块中的内容来确保执行关闭或清理资源的逻辑**

**理解中断**

调用静态方法Thread.interrupted() 对当前线程的中断标志位进行复位，如果该线程已经处于终结状态，即使该线程被中断过，在调用该线程对象的isInterrupted()依旧会返回false。

例如Thread.sleep(long millis)方法）这些方法在抛出InterruptedException之前，Java虚拟机会先将该线程的中断标识位清除，然后抛出InterruptedException，此时调用isInterrupted()方法将会返回false。 

通过标识位或者中断的方式能够使线程在终止时有机会去清理资源，而不是武断地将线程停止。

### 线程间通信

**Thread.join()** 线程A执行了Thread.join()，当前线程A等待Thread线程终止后才能从thread.join()返回

ThreadLocal ：以ThreadLocal对象为键、任意对象为值的存储结构。

### 线程应用实例 

**等待超时模式**

```Java
public synchronized Object get(long mills) throws InterruptedException{
    long future = System.currentTimeMillis() + mills;
    long remaining = mills;
    while((result == null) && remaining > 0){
        wait(remaining);
        remaining = future - System.currentTimeMillis();
    }
    return result;
    
}
```

针对昂贵资源的获取，都能加超时时间进行限制。

**线程池**

线程池的本质就是使用了一个线程安全的工作队列连接工作者线程和客户端线程，客户端线程将任务放到工作队列中便返回，而工作者线程则不断地从工作队列中取出工作并执行。

## 第五章 Java中的锁

### Lock接口

**Lock接口 提供的synchronized关键字不具备的主要特性**

尝试非阻塞地获取锁

能被中断地获取锁：当获取到的锁的线程被中断时，中断异常将会被抛出，同时锁会被释放

超时获取锁

**Lock的API**

lock  lockInterruptibly  tryLock tryLock(long time, TimeUnit unit)   unlock   newCondition()

### 队列同步器AQS

同步器的设计是基于模板方法模式的，使用者需要继承同步器并重写指定的方法。

同步器提供的模板方法基本分为3类：独占式获取与释放同步状态、共享式获取与释放同步状态和查询同步队列

