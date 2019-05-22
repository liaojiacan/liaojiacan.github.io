title: JDK1.7 ConcurrentHashMap的源码解读
author: Jiacan Liao
tags:
  - J.U.C
  - ''
  - ConcurrentHashMap
  - ''
categories:
  - 源码解读
  - JDK
date: 2019-02-26 11:18:00
---
### 一、 ConcurrentHashMap的数据结构(JDK7)。

1. segments[] : Segment<K,V> extends ReentrantLock 

> **分段锁**，HashMap 用一个Entry[] table 去存数据，ConcurrentHashMap 则是将 这个table 拆分出 n 个段（一个最接近concurrencyLevel的2的幂）分别存储，Segment 中的 用一个HashEntry table[] 来存数据，table中hash冲突的解决算法基本与HashMap一致。同一个段的put和get操作是需要加锁的，Segment继承了ReentrantLock 故有了锁的功能。

2. concurrencyLevel : int

> **并发等级**，默认是16，可以在构造函数指定该值，这个值直接影响segment数组的大小。如果这个值不是2的幂，则会计算出一个最接近（向上取）的2的幂来初始化segments数组。

3. segmentMask : int

> **掩码**，是一个bit位都是1的数，跟segments的长度有关，比如默认segments的长度是16=2的4次方（二进制为10000）。假如我们需要获取到一个数落在[0,16) 这个区间，则只需要用这个数跟1111做与运算, 得到的结果肯定是落在0到16之间，这个比取模运算更加高效。

4. segmentShift : int

> **位移数**，获取高ssize(segments size)位需要的左移的位数（32-ssize），hash函数算出来的是一个32 位int的整型，ConcurrentHashMap	对segments的hash算法采用的是一个取高位进行hash的做法。比如一个key算出来的值为1024，如果我想取高ssize位 ，假如ssize为4，那么就要将1024>>>(32-4)，取得高4位。获取到高4位后会与segmentMask进行与运算获取到一个[0,ssize)的数。这就是ConcurrentHashMap中对segment采用的hash算法。

* 为什么要采用高位运算？

> 源码中似乎没有说明，我猜是为了跟segment中的HashEntry[] table 的hash算法区分开来，降低冲突的概率。假如采用同样的hash算法，有2个key Hash到同一个segment中那么再进行 段中的二次hash的时候可能还是命中到同一个节点导致链越来越长。
```
 // segment[] 的hash算法 （hash的高位参与运算）
  int j = (hash >>> segmentShift) & segmentMask;
 // table[] 的hash算法 （hash的低位参与运算）
  int index = (tab.length - 1) & hash;
```

![image](https://github.com/liaojiacan/assets/blob/master/issue/ConcurrentHashMap_jdk1.7.png?raw=true)

### 二、segment中独占锁的加锁逻辑

 >分段锁的目的就是将锁冲突分离开，只有hash到同一个segment中的操作才会存在锁竞争，CurrentHashMap 中put和remove以及size是有加锁操作的。
 
 put操作加锁
 ```
 HashEntry<K,V> node = tryLock() ? null :scanAndLockForPut(key, hash, value);
```
 reomve操作加锁
 ```
    if (!tryLock())
       scanAndLock(key, hash);
 ```
 如果 tryLock() 不能能加锁成功则进行自旋，scanAndLockForPut和scanAndLock有点区别但是逻辑差不多。
 
 1.有限重试次数，多核心CPU的话是64次，单核1次，超过次数则阻塞等待获取锁。
 
 2.获取锁之前和获取到锁期间头节点不能发生改变，否则需要从头开始重试。
 ```
private void scanAndLock(Object key, int hash) {
    // similar to but simpler than scanAndLockForPut
    HashEntry<K,V> first = entryForHash(this, hash);
    HashEntry<K,V> e = first;
    int retries = -1;
    while (!tryLock()) {
        HashEntry<K,V> f;
        if (retries < 0) {
            if (e == null || key.equals(e.key))
                retries = 0;
            else
                e = e.next;
        }
        else if (++retries > MAX_SCAN_RETRIES) {
            lock();
            break;
        }
        // 如果头节点发生改变，从头开始扫描
        else if ((retries & 1) == 0 &&
                 (f = entryForHash(this, hash)) != first) {
            e = first = f;
            retries = -1;
        }
    }
}
 ```
 * get没有加排他锁，是否有线程安全问题？
 
先说下结论，ConcurrentHashMap get方法不存在线程安全问题，他的线程安全是由CAS 和 "volatile"保证的:

    1. UNSAFE.putOrderedObject/UNSAFE.getObjectVolatile
    2. volatile HashEntry<K,V> next;
    3. volatile V value;
    
我们整理下，要保证get不发生线程安全问题需要保证什么？

    1. get操作和put或remove操作并行的时候，get能操作能够获取到正确的segment和头节点table[i]。
    2. 在entries的遍历中能顺利走到未节点。
    3. 在1和2的前提下get操作时能够保证value值的可见性。

我们先看第1点时怎么保证的，我们都知道java中有个volatile用了保证变量在多线程下的可见性，volatile可以保证可见性，但是不能保证线程安全，如果当前赋值语句依赖当前值时是线程不安全的，比如a +=1 这种操作就是不安全的，显然在ConcurrentHashMap的并不需要这种操作，只存在简单的引用赋值操作。

但是需要注意的是一点，volatile修饰引用型变量时，只能保证当前引用的可见性，对于引用对象的内部变量仍然是无法保证可见性的，这就是为什么在对segments[] 数组和table[] 数组的的操作需要借助Unsafe类，而不是直接segments[i] = new Segment(...);

由前面的分析看，volatile/Unsafe.getObjectVolatile/Unsafe.putOrderedOject保证链当get操作晚与put操作时是可以获取到刚插入的节点(作为一个新头节点连接到旧节点并更新table)，对与一个早于put操作的get操作一个情况就是新插入的元素表头，但是get操作已经获取到了旧表头，所以并不影响get操作进行链表的遍历查找。

我们在看进行remove时是否会影响entries的遍历，从源码中看，HashEntry中的next成员是被volatile修饰的，这就保证了get可以安全得遍历到未节点。


### 三、size的实现逻辑

&emsp;&emsp;假如ConcurrentHashMap采用HashMap维护一个全局的size来变量统计大小，那么为了线程安全，也必定得改用原子类AtomicLong或者全局加锁。这显然与分段锁的设计背离。那么有没有一种比较折衷的办法呢？

&emsp;&emsp;ConcurrentHashMap中将size的统计拆分到各个segment取去护，每次执行size的时候将每个segment的count加起来，最终得到的结果就是map的大小。这个看似乎很合理，但是如果在进行统计的过程中有一个segment发生put或者remove操作呢，这样得到的结果就是错误的，显然我们可以在统计前先将每个segment给锁起来，再sum，得到的结果肯定是正确的。

&emsp;&emsp;**存在一种情况就是你的程序中并发很少，出现并发更新的情况很少，这个时候你执行size的时候将所有的segment加锁和不加锁的情况可能得到的结果是一样的，因为这个时候没有其他线程进行修改。似乎我们可以乐观地考虑一下大部分情况下是不需要进行锁操作的。**

&emsp;&emsp;Doug Lea采用类一种跟JDK集合类中大多数存在的fail-safe错误检查机制，对在每个segment中于更新操作维护一个modCount来记录更新的次数，统计前和统计后的modCount是一样的说明没有发生变化，当前的统计结果有效。ConcurrentHashMap的size方法的实现逻辑如下：

 - 先采用无锁的方式统计2次，如果前后的modCount总和是一样的，此次统计结果有效，返回结果。
 - 假如前后的modCount总和不一样，第三次进行有锁的统计。
```
public int size() {
        // Try a few times to get accurate count. On failure due to
        // continuous async changes in table, resort to locking.
        final Segment<K,V>[] segments = this.segments;
        int size;
        boolean overflow; // true if size overflows 32 bits
        long sum;         // sum of modCounts
        long last = 0L;   // previous sum
        int retries = -1; // first iteration isn't retry
        try {
            for (;;) {
                //第三次进行上锁
                if (retries++ == RETRIES_BEFORE_LOCK) {
                    for (int j = 0; j < segments.length; ++j)
                        ensureSegment(j).lock(); // force creation
                }
                sum = 0L;
                size = 0;
                overflow = false;
                for (int j = 0; j < segments.length; ++j) {
                    Segment<K,V> seg = segmentAt(segments, j);
                    if (seg != null) {
                        sum += seg.modCount;
                        int c = seg.count;
                        if (c < 0 || (size += c) < 0)
                            overflow = true;
                    }
                }
                // 前后2次的统计结果一致，可以返回
                if (sum == last)
                    break;
                last = sum;
            }
        } finally {
            if (retries > RETRIES_BEFORE_LOCK) {
                for (int j = 0; j < segments.length; ++j)
                    segmentAt(segments, j).unlock();
            }
        }
        return overflow ? Integer.MAX_VALUE : size;
    }


```

### 四、Unsafe.getObjectVolatile/Unsafe.putOrderedOject对偏移量的计算问题

&emsp;&emsp;ConcurrentHashMap中使用量Unsafe类来对segment数组和table数组进行数组填充和取值操作，其中对位置i的内存偏移计算用了位运算来代替乘法运算
```
// segment[0] 的偏移地址
int baseOffset = UNSAFE.arrayBaseOffset(Segment[].class);
// 每个位置的大小
int indexScale = UNSAFE.arrayIndexScale(Segment[].class);
//那么第i个元素的内存偏移就是
long offset = baseOffset+i*indexScale ;

```
上面的计算方法是利用乘法来计算的，但是乘法的计算还是比较慢的，如果能用位运算更佳。由于jvm给对象分配内存的时候会进行内存对对齐，也就是说indexScale其实会是一个2的n次方的数。一个整数i乘以一个2的n次方可以转化成 i<<n;

```
3 * 2 = 3 << 1
3 * 4 = 3 << 2
3 * 8 = 3 << 3
3 * 16 = 3 << 4
3 * 32 = 3 << 5
3 * 64 = 3 << 6
3 * 128 = 3 << 7
3 * 256 = 3 << 8
```

所以你会看到ConcurrentHashMap中有这样的代码
```
// 31 - Integer.numberOfLeadingZeros(ssize) 这个是求一个数x的2对数 
 SSHIFT = 31 - Integer.numberOfLeadingZeros(ssize);
 ...
 // 所以元素i在内存中的偏移就是
 long offset = SBASE +(i<<SSHIFT)
 
```

[测试用例-ConcurrentHashMap中利用Unsafe进行数组操作的测试用例](https://github.com/liaojiacan/code-snippets/blob/master/java-language/src/main/java/com/github/liaojiacan/unsafe/UnsafeArrayOperationTests.java);

