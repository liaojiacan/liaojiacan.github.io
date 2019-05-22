title: Java中的多线程和锁实现原理
author: Jiacan Liao
tags:
  - 多线程
  - synchronized
  - AQS
categories:
  - jdk
date: 2019-03-11 17:19:00
---
## 线程的实现
Java 规范里面并没有规定JVM要如何实现线程模型，在HotSpot VM 中使用的是1:1的线程模型，即1个java线程对应一个OS的线程（内核线程），在Thread中又很多native方法，就是调用OS的函数进行用户线程和内核线程的绑定。

- 每个线程都又一个内核线程与之绑定，用户线程推出，内核线程也会一起退出。
- 内核线程的数量是有限制的
- 内核线程调用，上下文切换开销很大。
### 线程调度
#### 线程的状态 （Thread.State枚举）
- NEW : 
- RUNNABLE : 对应的就绪和运行态
- BLOCKED : 阻塞状态，处于阻塞状态的线程会不断地请求资源，请求成功后就会进入就绪状态。
- WAITING : 等待状态，当线程调用wait,join,park等函数。等待状态下会释放资源，让出CPU和释放锁。需要其他线程唤醒。
- TIMED_WAITING  有限的等待。
- TERMINATED


### 线程相关的一些文章
- [Java的线程管理器能保证每个线程都有执行的机会么?](https://www.zhihu.com/question/27491155/answer/36847691)
- [wait/notify实现原理](http://www.hainiubl.com/topics/29)
## 锁

因为多线程的共享数据存在线程安全问题，需要通过一些控制来保证共享数据的读写，JVM层面提供sychronized的锁，而java层面current包下面有许多基于AQS的Lock的实现,在jdk1.6后,synchronized 和 ReentrantLock性能上以及没有太大的差距，ReentrantLock的使用更佳灵活，性能稳定，支持超时机制等，而采用synchronized不需要程序自己控制锁的加锁和释放，不容易出现死锁等问题。

### synchronized 的实现原理
JVM规范规定基于进入和退出monitor对象来控制方法和代码块的同步，也是就是monitorenter和monitorexit两个指令，当程序执行到monitorenter指令时会尝试获取对象的monitor所有权，也就是获取对象的锁。在最开始的JVM实现中是采用重量级锁的实现，线程的切换都涉及到用户态到内核态的切换，比较消化资源，所以在jdk1.6对锁进行优化。

#### 同步原理
> - JVM是怎么控制多线程程序的交替访问的？

Java中每个对象都有一个内置锁与之对应，所有需要对该对象进行排他性或者一致性访问时需要获取对象的内置锁（synchronized 中的代码，monitorenter指令）。这个内置锁的信息存在对象的对象头中（一些基本信息，其他的condition，队列等是在native heap中的）。一个对象的Monitor只能被一个线程获取到，其他线程得等待持有的Monitor的线程释放。
> 在一些官方的注释中说的是ObjectMonitor是一个内联锁对象的封装，就好比JVM层面实现的一个类似JUC框架下的Lock（不是说ObjectMonitor是JUC的Lock实现，说的是他们可能实现思路是一样的）。

做好线程的同步协调，我认为需要这3样东西（ObjectMonitor 和J.U.C的AQS 都是这样的）：
1. 维护一个竞争的互斥量
2. 一个队列
3. 线程的挂起和唤醒
> 实现同步也可以只用一个互斥量，自旋锁就是这么实现的，但是锁竞争太激烈会导致CPU做无用功。

想继续了解ObjectMonitor的实现可以看这几篇文章：
- [synchronized 与 object's Monitor](http://moonfacex.github.io/blog/java/2016/03/31/synchronized_and_monitor.html)
- [Moniter实现原理](https://www.hollischuang.com/archives/2030)

#### 对象头
> - Object的锁信息是存在在哪里的？
> - 在获取对象的锁的过程中都用到了对象头的哪些数据？

锁的信息存在java对象头里面。如果对象是数组，这虚拟机会用3个Word(32位虚拟机，32bit)来存对象头，如果对象是非数组类型，则用2个Word来存对象头，其中 有一个word用来存储对象的hashcode和锁信息，32bit，叫Mark word。
- Mark Word 不是一个固定的数据结构，具体的信息分布需要先判断2bit的锁标志位，不同的锁标志位，剩余的30bit可能表示不同的意思。
- 32bit的信息是不够存Monitor线程同步（调度）所需要的信息的，所以重量级锁是有另外的native heap存储的，之后再把指针存在Mark word 中。 
<table width="500" border="1" cellspacing="0" cellpadding="0">
<tbody>
<tr>
<td rowspan="2" valign="top" width="76"><strong>锁状态</strong></td>
<td colspan="3" valign="top" width="106">
<p align="center">25 bit</p>
</td>
<td rowspan="2" valign="top" width="85">
<p align="center">4bit</p>
</td>
<td valign="top" width="85">1bit</td>
<td valign="top" width="78">2bit</td>
</tr>
<tr>
<td colspan="2" valign="top" width="70">23bit</td>
<td valign="top" width="56">2bit</td>
<td valign="top" width="120">是否是偏向锁</td>
<td valign="top" width="100">锁标志位</td>
</tr>
<tr>
<td valign="top" width="90">轻量级锁</td>
<td colspan="5" valign="top" width="276">指向栈中锁记录的指针</td>
<td valign="top" width="78">00</td>
</tr>
<tr>
<td valign="top" width="76">重量级锁</td>
<td colspan="5" valign="top" width="276">指向互斥量（重量级锁）的指针</td>
<td valign="top" width="78">10</td>
</tr>
<tr>
<td valign="top" width="76">GC标记</td>
<td colspan="5" valign="top" width="276">空</td>
<td valign="top" width="78">11</td>
</tr>
<tr>
<td valign="top" width="76">偏向锁</td>
<td valign="top" width="80">线程ID</td>
<td colspan="2" valign="top" width="80">Epoch</td>
<td valign="top" width="120">对象分代年龄</td>
<td valign="top" width="85">1</td>
<td valign="top" width="78">01</td>
</tr>
</tbody>
</table>

#### 锁的优化
在jdk1.6之前synchronized是单纯的重量级锁实现，由于重量级锁，线程获取不到锁就需要挂起等待唤醒，这种切换涉及到了用户态到内核态的转换，开销还是比较大的。在jdk1.6加入了偏向锁、轻量级锁。只有一个线程请求对象锁的时候，启用的是偏向锁，当有第二个线程竞争的时候（应该说是偏向状态出现锁竞争），这个时候会升级为轻量级锁（cas 自旋锁），处于轻量级锁状态下，如果自旋10次（可以配置）还是获取锁失败，则锁升级为重量级锁。
- 偏向锁 ：在大部分情况下一个同步方法或者一个同步代码块不存在多线程的竞争，这样只需要在对象头和当前线程的栈帧中存一个线程ID，每次获取锁的时候只需要判断一些线程ID释放一致就行了，不用进行CAS的加锁和解锁。如果有第二个线程需要竞争锁，这个时候会通过CAS设置Mark Word中的锁状态位，成功则修改为偏向当前线程，失败的话就进行锁的升级，锁升级涉及到偏向锁的撤销，会将偏向锁线程挂起。
    > [偏向锁会将Mark Word设置为当前threadId，那么hashCode存哪里了?](https://stackoverflow.com/questions/14717736/where-is-objects-hash-code-stored-if-biased-locking-is-enabled-in-hotspot-jvm)
    > 如果处于偏向的的对象调用的hashCode方法就会触发撤销偏向锁 
- 轻量级锁：线程在获取锁之前，当前线程会在栈帧中创建一个Mark Word的拷贝作为锁记录，官方称为Displaced Mark Word。然后将对象头中替换成锁记录的指针（CAS），如果失败则会自旋10次（在1.6之后是采用自适应锁，这个时间已经不能自己配置了），之后升级为重量级锁。
    > 为什么一定要拷贝到Displaced Mark Word，而不直接就采用一个threadId？一个原因是需要恢复hash和GC分代的信息，一个就是解决重入锁的问题。

- 重量级锁：ObjectMonitor有更多的空间来实现线程同步，可以更好像的实现线程同步（挂起和唤醒）。[轻量级锁为什么要膨胀？](https://www.zhihu.com/question/41930877/answer/136699311)

#### 优缺点

锁 | 优点 | 缺点 | 使用场景
---|---|---|---
偏向锁 | 只需比较threadId释放是否是当前线程<br>没有CAS的消耗 | 当出现锁竞争的时候会有锁撤销的消耗 | 单个线程
轻量级锁 | 线程一直在用户态，不用挂起。没有线程切换的消耗| 自旋会导致CPU做无用功| 同步代码块执行较快。
重量级锁 | 线程挂起，不用进行自旋 | 用户态到内核态转化，开销大| 同步代码块执行时间较长，锁竞争激烈

### J.U.C中的锁
上面锁的synchronzied是JVM的内置锁，在1.6之前性能比较差，Doug Lea就写个并发框架(java.util.current)，在1.6之后synchronized的性能已经跟Lock查不不多了，但是还少了锁的获取和释放的操作性，不支持超时，只有一个condition等。
#### AQS
AbstractQueueSynchronizer是J.U.C中其他锁或者同步器的基础框架（ReentrantLock,ReentrantReadWriteLock,CountdDownLatch,CyclicBarrier等），这些框架在AQS的基础上进行了扩展，通常是继承AQS然后实现了AQS的几个抽象方法。
> 我在上面说ObjectMonitor有说到，同步器为了完成同步工作，需要3个东西：
1. 用于同步的状态量
2. 一个队列或者一个保存等待线程的容器
3. 线程的挂起和唤醒

我们看下AQS是怎么围绕这3个部分进行实现的。

##### 1. 同步的状态量(互斥量)
AQS中维护一个volatile 的int 变量state，线程通过cas来获取这个互斥量。AQS提供一下几个方法来对state变量进行操作。
- getState()
- compareAndSetState(int expect, int update)
- setState(int state)

有了上面的3个方法，同步器就可以实现自旋锁，但是如果想实现公平锁，上面的三个方法或者说单用一个state变量是无法做到了。这个时候就需要一个FIFO的队列来维护这些线程。此外为了实现重入锁，我们还得需要一个变量来存当前持有的锁是什么线程。

##### 2.等待线程队列
AQS 用了一个CLH的双向队列，Node的数据结构大概如下：
```
statci final class Node{
    
    volatile int waitStatus;
    volatile Node prev;
    volatile Node next;
    Node nextWaiter;
}

```
AQS维护一个头节点和一个尾节点，入队的时候通过CAS加入到未尾节点中。入队后开始开始自旋。
```
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            // 前趋节点是头节点，并且获取到互斥量，说明获取锁成功。
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 判断是否需要挂起
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
从上面代码片段可以看出，加入队列的线程节点并不是完全的自旋，shouldParkAfterFailedAcquire方法判断当前线程是否需要挂起。
> 下面的这个方法表明shouldParkAfterFailedAcquire 会在调用1到2次后会返回true（如果期间节点没有发生改变的话）。也就是自旋锁只自旋了2次就会被挂起。
```
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /* 
             * 进行到这里说明前驱节点的waitStatus 是0 或者PROPAGATE ，利用CAS的设置为SIGNAL，这样下次自旋就会阻塞了，这里不返回true的目的是让当前线程再自旋一次，确保挂起前是无法获取到锁（避免发生刚挂起就被唤醒的情况）。
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```

##### 3.线程的挂起和唤醒
AQS中实现线程的挂起和唤醒是通过LockSupport这个工具，LockSupport的底层实现是调用Unsafe的native方法。
```
public static void unpark(Thread thread) {
    if (thread != null)
        UNSAFE.unpark(thread);
}

public static void park(Object blocker) {
    Thread t = Thread.currentThread();
    setBlocker(t, blocker);
    UNSAFE.park(false, 0L);
    setBlocker(t, null);
}
```

##### ConditionObject
ConditionObject 是AQS实现类似object类的wait/notify/notifyAll方法的，ConditionObject提供的是aw
ait/awaitNanos(long nanos)/awaitUtil(Date date)/awaitUniterrutibly()/signal()/signalAll()。底层的实现也是各自维护一个队列，Node.nextWaiter。

- 对于超时机制也是用LockSupport中的实现，但并不是所有情况下都使用系统的休眠，有个休眠的自旋时间阀值`spinForTimeoutThreshold = 1000L` ，默认是1000 纳秒，少于这个阀值的都不用休眠，而是直接自旋。


### 参考文章
- [synchronized 与 object's Monitor](http://moonfacex.github.io/blog/java/2016/03/31/synchronized_and_monitor.html)
- [AbstractQueuedSynchronizer的介绍和原理分析](http://ifeve.com/introduce-abstractqueuedsynchronizer/)
- [J.U.C之AQS：阻塞和唤醒线程](https://blog.csdn.net/chenssy/article/details/65449785)
- [关于synchronized的Monitor Object机制的研究](https://blog.csdn.net/m_xiaoer/article/details/73274642)
- [Intrinsic Locks and Synchronization](https://docs.oracle.com/javase/tutorial/essential/concurrency/locksync.html)