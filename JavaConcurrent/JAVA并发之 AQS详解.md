[++博客原文++](https://www.cnblogs.com/waterystone/p/4920797.html) 

**概述**： AQS定义了一套多线程访问共享资源的同步器框架，许多同步实现都依赖于它，比如ReeentrantLock/CountDownLatch

## 框架

![++框架++](https://images2015.cnblogs.com/blog/721070/201705/721070-20170504110246211-10684485.png)

它维护了一个**volatile int state**（代表共享资源）和一个**FIFO线程等待队列**（多线程争用资源被阻塞时会进入此队列）。这里volatile是核心关键词，具体volatile的语义，在此不述。state的访问方式有三种:getState() setState() compareAndSetState()   
AQS定义两种资源共享方式：**Exclusive**（独占，只有一个线程能执行，如ReentrantLock）和**Share**（共享，多个线程可同时执行，如Semaphore/CountDownLatch）。

不同的自定义同步器争用共享资源的方式也不同。**自定义同步器在实现时只需要实现共享资源state的获取与释放方式即可**，至于具体线程等待队列的维护（如获取资源失败入队/唤醒出队等），AQS已经在顶层实现好了。自定义同步器实现时主要实现以下几种方法：   
`1.`isHeldExclusively()：该线程是否正在独占资源。只有用到condition才需要去实现它。   
`2.`tryAcquire(int)：独占方式。尝试获取资源，成功则返回true，失败则返回false。   
`3.`tryRelease(int)：独占方式。尝试释放资源，成功则返回true，失败则返回false。    
`4.`tryAcquireShared(int)：共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。    
`5.`tryReleaseShared(int)：共享方式。尝试释放资源，如果释放后允许唤醒后续等待结点返回true，否则返回false。   

以ReentrantLock为例，state初始化为0，表示未锁定状态。A线程lock()时，会调用**tryAcquire()**独占该锁并将**state+1**。此后，其他线程再tryAcquire()时就会失败，直到A线程unlock()到**state=0**（即释放锁）为止，其它线程才有机会获取该锁。当然，释放锁之前，A线程自己是可以重复获取此锁的（state会累加），这就是**可重入**的概念。但要注意，**获取多少次就要释放多么次**，这样才能保证state是能回到**零态**的。

再以CountDownLatch以例，任务分为N个子线程去执行，state也初始化为N（注意N要与线程个数一致）。这N个子线程是并行执行的，每个子线程执行完后countDown()一次，**state会CAS减1**。等到所有子线程都执行完后(即state=0)，会unpark()主调用线程，然后主调用线程就会从await()函数返回，继续后余动作。

## 独占模式下源码详解

### 1. acquire(int)
此方法是独占模式下线程获取共享资源的顶层入口。如果获取到了资源，线程直接返回，否则进入到等待队列中，直到获到资源为止，***整个过程是忽略中断的影响。***

    public final void acquire(int arg) {
      if (!tryAcquire(arg) &&
          acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
          selfInterrupt();
    }

函数流程如下：  
1.tryAccquire() 尝试直接去获取资源，成功后直接返回。  
2.addWaiter() 将该线程加入到等待队列尾部，并标记为独占模式。  
3.accquireQueued()使线程在等待队列中一直获取资源后才返回。如果在整个等待过程中被中断过，则返回true,否则返回false。  
4.如果线程在等待过程中被中断过，它是不进行响应的。只有在获取到资源之后才进行自我中断selfInterrupt()，将中断补上。

---------
待补充   
--------- 

对tryAccquireQueued()的补充
1.节点进入到队列尾部后，需要检查状态， 找到安全的休息点   
2.调用park()进入到waiting状态，等待unpark() 或者 interrupt() 唤醒自己   
3.被唤醒后，看看自己有没有资格拿到号，如果拿到了，就让head指向当前节点，并返回从入队到拿到号整个过程中是否被中断过；没拿到的话，继续1

![++流程图++](https://images2015.cnblogs.com/blog/721070/201511/721070-20151102145743461-623794326.png)

### 2.release(int)
反操作release，此方法是在独占模式下线程释放共享资源的顶层入口。它会释放指定量的资源，如果彻底释放了（state = 0），它会唤醒等待队列中其他线程来获取资源。

    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;//找到头结点
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);//唤醒等待队列里的下一个线程
            return true;
        }
        return false;
    }



### accquireShared()
这里tryAcquireShared()依然需要自定义同步器去实现。但是AQS已经把其返回值的语义定义好了：负值代表获取失败；0代表获取成功，但没有剩余资源；正数表示获取成功，还有剩余资源，其他线程还可以去获取。所以这里acquireShared()的流程就是：      
1. tryAccquiredShared() 尝试获取资源， 成功后直接进行返回   
2. 失败则通过doAccquireShared()进入到等待队列，直到unpark()/interrupted() 成功获取到资源才能够返回。整个等待过程也是忽略中断的。
3. 其实跟accquire 的流程差不多，只不过是自己拿到了资源，还会去***唤醒队友们的操作***

### releasedShared()
反操作releaseShared()吧。此方法是共享模式下线程释放共享资源的顶层入口。它会释放指定量的资源，如果成功释放且允许唤醒等待线程，它会唤醒等待队列里的其他线程来获取资源。

    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {//尝试释放资源
            doReleaseShared();//唤醒后继结点
            return true;
        }
        return false;
    }


此方法的流程也比较简单，一句话：释放掉资源后，唤醒后继。跟独占模式下的release()相似，但有一点稍微需要注意：独占模式下的tryRelease()在完全释放掉资源（state=0）后，才会返回true去唤醒其他线程，这主要是基于独占下可重入的考量；而共享模式下的releaseShared()则没有这种要求，共享模式实质就是控制一定量的线程并发执行，那么拥有资源的线程在释放掉部分资源时就可以唤醒后继等待结点。例如，资源总量是13，A（5）和B（7）分别获取到资源并发运行，C（4）来时只剩1个资源就需要等待。A在运行过程中释放掉2个资源量，然后tryReleaseShared(2)返回true唤醒C，C一看只有3个仍不够继续等待；随后B又释放2个，tryReleaseShared(2)返回true唤醒C，C一看有5个够自己用了，然后C就可以跟A和B一起运行。而ReentrantReadWriteLock读锁的tryReleaseShared()只有在完全释放掉资源（state=0）才返回true，所以自定义同步器可以根据需要决定tryReleaseShared()的返回值。


### 小结
本节我们详解了独占和共享两种模式下获取-释放资源(acquire-release、acquireShared-releaseShared)的源码，相信大家都有一定认识了。值得注意的是，acquire()和acquireSahred()两种方法下，线程在等待队列中都是忽略中断的。AQS也支持响应中断的，acquireInterruptibly()/acquireSharedInterruptibly()即是，这里相应的源码跟acquire()和acquireSahred()差不多，这里就不再详解了。


### 互斥锁（Mutex）
Mutex是一个不可重入的互斥锁实现。锁资源（AQS里的state）只有两种状态：0表示未锁定，1表示锁定。

### 来自WHL的总结
AQS框架下ReentrantLock执行lock过程：  
1. tryAcquire() 尝试获取资源，如果成功直接返回
2. addWaiter() 添加到等待队列的尾部，标记为独占模式 
3. acquireQueued() 使线程在等待队列找到安全点后，park等待休息，轮到自己(unpark()或者interrupt) 唤醒，唤醒后会去尝试获取资源，获取资源后返回，如果在整个过程中被中断过，返回true
4. 线程在等待过程中被中断过，是不响应的，只是获取资源后再进行自我中断selfInterrupt(),将中断补上。


release()是根据tryRelease()的返回值判断线程是否已经完成资源的释放。 unparkSuccessor() 方法来唤醒队列中下一个不放弃的线程。
***用unpark()唤醒等待队列中最前边的那个未放弃线程，这里我们也用s来表示吧。此时，再和acquireQueued()联系起来，s被唤醒后，进入if (p == head && tryAcquire(arg))的判断（即使p!=head也没关系，它会再进入shouldParkAfterFailedAcquire()寻找一个安全点。这里既然s已经是等待队列中最前边的那个未放弃线程了，那么通过shouldParkAfterFailedAcquire()的调整，s也必然会跑到head的next结点，下一次自旋p==head就成立啦），然后s把自己设置成head标杆结点，表示自己已经获取到资源了，acquire()也返回了***
