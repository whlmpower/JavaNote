[ref:参考博客](http://www.cnblogs.com/cm4j/p/juc_condition.html)    
JUC提供了Lock方便的进行锁操作，有时候我们需要对线程进行条件性的阻塞和唤醒，需要condition，方便的对持有锁的线程进行阻塞和唤醒。   
# Condition的概念
JDK官方解释：   
条件（也称为条件队列 或条件变量）为线程提供了一个含义，以便在某个状态条件现在可能为 true 的另一个线程通知它之前，一直挂起该线程（即让其“等待”）。因为访问此共享状态信息发生在不同的线程中，所以它必须受保护，因此要将某种形式的锁与该条件相关联。等待提供一个条件的主要属性是：以原子方式 释放相关的锁，并挂起当前线程，就像 Object.wait 做的那样。   

AQS有一个队列，Condition也有一个队列，两者是相对独立的队列，因此**一个lock可以有多个condition**，Lock(AQS) 的队列也是阻塞线程，Condition队列也是阻塞线程，但是它有阻塞和解除阻塞的功能。   
## 大体实现流程
await() 就是在当前线程持有锁的基础上释放资源，并建立Condition节点加入到Condition的队列尾部，阻塞当前线程   
signal() 将Condition的头结点移动到AQS等待节点的尾部，让其等待再次获取锁   
示例图：   
![++初始状态++](https://images0.cnblogs.com/blog/196062/201308/27181155-b1e82b053f9240069be617af187a6ff5.png)

**节点1执行Condition.await()**   
1. 将head后移  
2. 节点1释放锁并从AQS队列中移除  
3. 将节点1加入到Condition的等待队列中   
4. 更新lastWaiter为节点1   

![image](https://images0.cnblogs.com/blog/196062/201308/27181203-de3e3e892c0741f48b1790d73c339288.png)

**节点2 执行signal()操作**    
5. 将firstWaiter后移  
6. 将节点4移除Condition队列 
7. 将节点4加入到AQS等待队列中  
8. 更新AQS的等待队列的tail   

![image](https://images0.cnblogs.com/blog/196062/201308/27181209-5b76222da54f4df7ada76277b7a392c6.png)

## Condition 的数据结构 

我们知道一个Condition可以在多个地方被await()，那么就需要一个FIFO的结构将这些Condition串联起来，然后根据需要唤醒一个或者多个（通常是所有）。所以在Condition内部就需要一个FIFO的队列。   
private transient Node firstWaiter;   
private transient Node lastWaiter;   
上面的两个节点就是描述一个FIFO的队列。我们再结合前面提到的节点（Node）数据结构。我们就发现Node.nextWaiter就派上用场了！nextWaiter就是将一系列的Condition.await*串联起来组成一个FIFO的队列。  
----
**线程何时阻塞和释放**  
**阻塞：**await()方法中，在线程释放资源后，如果节点**不在AQS**等待队列中，则**阻塞**当前线程，如果在等待队列中，则自旋尝试尝试获取锁   
**释放：**signal()后，节点会从condition 队列**移动到AQS**等待队列中，进入正常的锁获取流程。
----

## await方法  
await()操作实际上就是释放锁，然后挂起线程，一旦条件满足就被唤醒，再次获取锁！
 
 ```java
 public final void await() throws InterruptedException {
    // 1.如果当前线程被中断，则抛出中断异常
    if (Thread.interrupted())
        throw new InterruptedException();
    // 2.将节点加入到Condition队列中去，这里如果lastWaiter是cancel状态，那么会把它踢出Condition队列。
    Node node = addConditionWaiter();
    // 3.调用tryRelease，释放当前线程的锁
    long savedState = fullyRelease(node);
    int interruptMode = 0;
    // 4.为什么会有在AQS的等待队列的判断？
    // 解答：signal操作会将Node从Condition队列中拿出并且放入到等待队列中去，在不在AQS等待队列就看signal是否执行了
    // 如果不在AQS等待队列中，就park当前线程，如果在，就退出循环，这个时候如果被中断，那么就退出循环
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    // 5.这个时候线程已经被signal()或者signalAll()操作给唤醒了，退出了4中的while循环
    // 自旋等待尝试再次获取锁，调用acquireQueued方法
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null)
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
 ```
 整个await过程如下：  
 1. 将当前线程加入Condition锁队列，这里不同于AQS，进入的是Condition FIFO队列  
 2. 释放锁。
 3. 自旋挂起，阻塞，看自己线程是否在AQS等待队列，不在的话，阻塞。
 4. 被唤醒或者超时或者Cacelled，将自己从Condition中移除，加入到AQS队列，并且再次尝试获取锁(调用accquireQueued)     
 
## singal 和 singalAll 方法
按照signal/signalAll的需求，就是要将Condition.await\*()中FIFO队列中第一个Node唤醒（或者全部Node）唤醒。尽管所有Node可能都被唤醒，但是要知道的是仍然**只有一个线程能够拿到锁**，其它没有拿到锁的线程仍然需要自旋等待，就上上面提到的第4步(acquireQueued)。 
 
```Java
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}
```
这里先判断当前线程是否持有锁，如果没有持有，则抛出异常，然后判断整个condition队列是否为空，不为空则调用doSignal方法来唤醒线程，看看doSignal方法都干了一些什么：  
```Java
private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
```
上面的代码很容易看出来，signal就是唤醒Condition队列中的第一个非CANCELLED节点线程，而signalAll就是唤醒所有非CANCELLED节点线程。当然了遇到CANCELLED线程就需要将其从FIFO队列中剔除。   
```Java
final boolean transferForSignal(Node node) {
    /*
     * 设置node的waitStatus：Condition->0
     */
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    /*
     * 加入到AQS的等待队列，让节点继续获取锁
     * 设置前置节点状态为SIGNAL
     */
    Node p = enq(node);
    int c = p.waitStatus;
    if (c > 0 || !compareAndSetWaitStatus(p, c, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```
上面就是唤醒一个await\*()线程的过程，根据前面的介绍，如果要unpark线程，并使线程拿到锁，那么就需要线程节点进入AQS的队列。所以可以看到在LockSupport.unpark之前调用了**enq(node)操作，将当前节点加入到AQS队列**。 

## Condition 中的中断
中断发生在await操作之前，此方法一定要抛出一个InterruptedException   
中断发生在await之后，方法不抛出异常，而是系统的中断状态集  
条件队列需要一个状态位，当出现Signal信号失败，就将信号传递到队列的下一个节点内，如果出现Cancel信号失败，就取消传递操作，唤醒锁的重新获取操作  

