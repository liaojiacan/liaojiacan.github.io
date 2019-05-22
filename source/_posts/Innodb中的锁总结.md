title: Innodb中的锁总结
author: Jiacan Liao
tags:
  - Innodb

categories:

  - 数据库
date: 2019-02-28 19:22:00
---
### 锁的种类

#### 表锁
- LOCK TABLE *table_name* READ : 用读锁锁表，阻塞其他事务修改
- LOCK TABLE *table_name* WRITE: 用写锁锁表，阻塞其他事务读和写

#### 行锁

- X锁：排他锁，允许对数据进行删除和更新/插入
- S锁：共享锁，允许对数据进行读取，可理解为读锁
> 锁的兼容性：如果两个事务能对一行数据同时加锁，就认为这个锁是兼容的，如果是要等待其他事务释放，则认为这2个锁是不兼容的。

 * | X | S
---|---|---
 X | 不兼容| 不兼容
S  | 不兼容 | 兼容

#### 意向锁

MySQL innodb 是支持多粒度锁的，比如可以同时存在表锁和行锁，为了更好的实现多粒度锁，innodb引入了意向锁，在innodb中的意向锁是表级的锁，意向锁与意向锁之间是兼容了，假如不存在表锁，不会有事务在加意向锁的时候阻塞。
- IS锁：意向共享锁，表明表中存在一行或者多行的S锁，即在给一行数据加S锁之前必在这个表加IS锁。
- IX锁：意向排他锁，表明表中存在一行或者多行的X锁，即在给一行数据加S锁之前必在这个表加IX锁。

> 思考：为什么需要这个意向锁？

我们比如思考一下这个场景，事务1给某个表中的N行加了行锁，这个时候事务2想给这个表加个表锁，那么事务2需要确认的事情有：
1. 这个表是否存在不兼容的表锁，比如我想加个X锁，但是已经有其他事务加了S锁。
2. 这个表中是否已经存在不兼容的行锁。

显然在确认第二个条件时，如果采用全表扫描的话，效率太低，所以意向锁的目的就是在粗粒度的锁（表锁）可以快速判断是否与低粒度的锁存在冲突。


| *  | IX       | S          | IS         |
| -- | -------- | ---------- | ---------- |
| X  | 不兼容 | 不兼容   | 不兼容   |
| IX | 不兼容 | 兼容 | 不兼容   |
| S  | 不兼容 | 不兼容   | 兼容 |
| IS | 不兼容 | 兼容 | 兼容 |

### innodb行锁的形式或者算法
> Innodb中行锁都是加在索引上的，针对不同的场景都有不同的加锁策略。

#### Record Lock
记录锁，顾名思义就是锁住记录本身，锁主主键和唯一索引，如果表中没有加任何索引，锁会加在隐式生成的主键上。

#### Gap Lock

间隙锁，当存在范围扫描的时候，给扫描范围加间隙锁，[起始地址，终止)
1. 不同事务对同一个区间加间隙锁是不冲突的，所以S Gap Lock 和 X Gap Lock不存在区别。
2. READ_COMMITED 隔离级别下不会启用间隙锁。

#### Next Key Lock
临键锁，Record Lock + Gap Lock ,锁主当前值区间+下一个区间(不一定)。

```
10, 11, 13, 20
(negative infinity, 10]
(10, 11]
(11, 13]
(13, 20]
(20, positive infinity)
```
1. 解决的是当前读下的幻读问题。

#### Insert Intention Lock
插入意向锁，是一种特殊的间隙锁；insert之前会向插入区间加上Insert Intention Lock。
1. Gap Lock / Next key Lock 与 Insert Intention Lock不兼容。
2. Gap lock 和Next key lock 的目的就是防止有数据插入间隙

### 不同SQL产生的锁
https://dev.mysql.com/doc/refman/5.7/en/innodb-locks-set.html
1. SELECT...FROM 
> 一致性非锁定读，除了在 SERIALIZABLE隔离级别下会存在S锁，其他隔离级别下都不会有锁。

2. SELECT...FOR UPDATE/SELECT...LOCK IN SHARE MODE
> - 也叫做当前读，会在给扫描过程中的索引加X或S锁，跟Where条件实际上没有强关系，只跟扫描的过程有关，所以where条件是否能够命中索引比较重要。
> - 在检索的过程中会给用到的索引加Next Key Lock。不过如果检索条件是唯一索引能定位到一行数据，则只加Record Lock

3. UPDATE...WHERE... 
> 同样给检索到的记录加next key lock, 如果WHERE条件中是使用主键或者唯一索引进行限定的话，只在索引加了Record Lock。
> 如果UPDATE的是聚簇索引记录，会对受影响的辅助索引加隐式锁，当有新的辅助索引插入前的重复检测，以及在执行插入新的辅助索引记录时，对受影响的索引记录加S锁。

4. DELETE FROM ... WHERE ... 
> 与UPDATE基本一致

5. INSERT
> - 给插入索引的记录加一个X锁
> - 插入前前会加个Insert Intention Lock。
> - 如果发生唯一键异常（duplicate-key error ），会在原记录上加S锁，这个如果和delete和update一起使用可能会导致死锁。

6. INSERT ... ON DUPLICATE KEY UPDATE 
> - 跟INSERT语句有点不同的就是，当发生重复键异常是，这里加的是排他锁，而不是共享锁。
> - 如果是唯一键异常，则加的是Next key lock。

7. REPLACE 
> - 如果没有发生冲突，则行为跟INSERT是一致的。
> - 如果发生冲突，则对唯一键加的是Next key lock。

8. INSERT INTO T SELECT ... FROM S WHERE ...
> - 给子查询语句加的X锁。
> - 如果是READ COMMITED 隔离基本，则采用的是快照读

9. 外键约束
> 在进行外键约束检测时，会给记录加行级共享锁。

### 死锁

> 死锁是只当2个或者2个以上的事务抢占各自的资源，导致的相互等待的现象。

#### 案例
1. [Insert-ignore-和update-导致的死锁问题分析](http://liaojiacan.me/2019/02/27/Insert-ignore-%E5%92%8Cupdate-%E5%AF%BC%E8%87%B4%E7%9A%84%E6%AD%BB%E9%94%81%E9%97%AE%E9%A2%98%E5%88%86%E6%9E%90/)

#### 解决死锁和进行死锁检测
1. 设置超时时间,事务有限时间超时,回滚其中一个事务。
```
mysql> show variables like "%innodb_lock_wait_timeout%";
+--------------------------+-------+
| Variable_name            | Value |
+--------------------------+-------+
| innodb_lock_wait_timeout | 50    |
+--------------------------+-------+
1 row in set (0.07 sec)
```
2. wait-graph 等待图死锁检测
- 锁的信息链
- 事务等待链

