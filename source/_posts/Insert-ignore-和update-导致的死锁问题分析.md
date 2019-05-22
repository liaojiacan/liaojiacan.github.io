title: Insert ignore 和update 导致的死锁问题分析
author: Jiacan Liao
tags:
  - MySQL
  - innodb
categories:
  - 死锁
date: 2019-02-27 20:13:00
---
### 业务逻辑以及死锁现象
业务逻辑大概如下：

1. 在粉丝表新增一条关系记录。
2. 假如关注者也是当前用户的粉丝，则更新2者的标记为相互关注。

业务代码如下, 最初是考虑用insert ignore 来解决幂等的问题，所以**允许重复调用**。
```

//是否 关注的用户是操作者的粉丝（互粉）
boolean isHisFans = fansDao.isFans(fansUserId,followingUserId);

//	insert ignore into 
//  fans(fans_user_id,following_user_id,following_each_other)
//	values(#{fansUserId}, #{followingUserId},#{followingEachOther})
int updateNum = fansDao.addFans(fansUserId, followingUserId,isHisFans);

boolean addAndCheck = false;

if(isHisFans){
    // update fans
	// set following_each_other = #{followingEachOther}
	// where (fans_user_id = #{fansUserId} and following_user_id =  #{followingUserId} ) or (fans_user_id = #{followingUserId} and following_user_id = #{fansUserId} )
	int num = fansDao.setFollowingEachOther(followingUserId,fansUserId,true);
	if( num >0 ){
		//重新调整数据
		addAndCheck = true;
	}
}

```

线上产生的死锁的信息(show engine innodb status)

```
LATEST DETECTED DEADLOCK
------------------------
2019-02-12 23:24:45 7fa406a4f700
*** (1) TRANSACTION:
TRANSACTION 515545684, ACTIVE 0 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1184, 2 row lock(s)
MySQL thread id 2121, OS thread handle 0x7fa406b12700, query id 1702086 10.10.17.63 jb-glive Searching rows for update
update fans
		set following_each_other = 1
		where (fans_user_id = '1776093' and following_user_id = '1331089' ) or (fans_user_id = '1331089' and following_user_id = '1776093' )
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 1023 page no 37913 n bits 400 index `UK_USER_ID` of table `glive`.`fans` trx id 515545684 lock_mode X locks rec but not gap waiting
Record lock, heap no 334 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 7; hex 31333331303839; asc 1331089;;
 1: len 7; hex 31373736303933; asc 1776093;;
 2: len 8; hex 8000000001282c31; asc      (,1;;

*** (2) TRANSACTION:
TRANSACTION 515545683, ACTIVE 0 sec starting index read
mysql tables in use 1, locked 1
5 lock struct(s), heap size 1184, 4 row lock(s)
MySQL thread id 3485, OS thread handle 0x7fa406a4f700, query id 1702085 10.10.17.61 jb-glive Searching rows for update
update fans
		set following_each_other = 1
		where (fans_user_id = '1776093' and following_user_id = '1331089' ) or (fans_user_id = '1331089' and following_user_id = '1776093' )
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 1023 page no 37913 n bits 400 index `UK_USER_ID` of table `glive`.`fans` trx id 515545683 lock_mode X locks rec but not gap
Record lock, heap no 334 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 7; hex 31333331303839; asc 1331089;;
 1: len 7; hex 31373736303933; asc 1776093;;
 2: len 8; hex 8000000001282c31; asc      (,1;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 1023 page no 38446 n bits 360 index `UK_USER_ID` of table `glive`.`fans` trx id 515545683 lock_mode X locks rec but not gap waiting
Record lock, heap no 294 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 7; hex 31373736303933; asc 1776093;;
 1: len 7; hex 31333331303839; asc 1331089;;
 2: len 8; hex 8000000001282c59; asc      (,Y;;

*** WE ROLL BACK TRANSACTION (1)
```
锁分析：

1. 事务T2持有 (fans_user_id=1331089,following_user_id=1776093,主键=hex8000000001282c31) X 锁；
2. 事务T2等待行锁
 (fans_user_id=1776093,following_user_id=1331089,主键=hex8000000001282c59) X 锁；
3. 事务T1等待T2持有的锁。
4. 事务T1此时应该还持有T2等待的锁，只是没显示出来。

> 看起来就是典型的AB-BA问题

## 重现

我们先简化上面的业务代码逻辑，假设fans_user_id=11000,following_user_id=10086，其实就是执行2个SQL：
```
1. insert ignore into fans(fans_user_id,following_user_id) values(11000,10086);
2. update fans set following_each_other = 1
		where (fans_user_id = '11000' and following_user_id = '10086' ) 
		or (fans_user_id = '10086' and following_user_id = '11000' );
```
场景一：不存在fans_user_id=11000,following_user_id=10086的这条数据。

T1 | T2
---|---
>begin; | >begin;
insert ignore into fans(fans_user_id,following_user_id)<br> values(11000,10086); | *
* | insert ignore into fans(fans_user_id,following_user_id)<br> values(11000,10086); //阻塞
update fans<br> set following_each_other = 1 <br> where (fans_user_id = '11000' and following_user_id = '10086' ) <br> or (fans_user_id = '10086' and following_user_id = '11000' ) | *
commit; | insert 语句开始执行
* | update fans<br> set following_each_other = 1 <br> where (fans_user_id = '11000' and following_user_id = '10086' ) <br> or (fans_user_id = '10086' and following_user_id = '11000' )
* | commit;

> 分析：
> 1. 由于数据库并没有这条记录，所以**事务T1**在执行insert ignore into 时可以执行成功，并给这行数据加了X锁。
> 2. **事务T2**在执行insert ignore into 时由于获取不到行锁，直接阻塞。
> 3. 后面都是顺序执行，所以并不会出现死锁的问题。

场景二：数据库已经存在fans_user_id=11000,following_user_id=10086的这条数据。

T1 | T2
---|---
>begin; | >begin;
insert ignore into fans(fans_user_id,following_user_id)<br> values(11000,10086); | *
* | insert ignore into fans(fans_user_id,following_user_id)<br> values(11000,10086); //执行成功
update fans<br> set following_each_other = 1 <br> where (fans_user_id = '11000' and following_user_id = '10086' ) <br> or (fans_user_id = '10086' and following_user_id = '11000' ) | *
阻塞等待 | *
* | update fans<br> set following_each_other = 1 <br> where (fans_user_id = '11000' and following_user_id = '10086' ) <br> or (fans_user_id = '10086' and following_user_id = '11000' )
* | Deadlock found when trying to get lock; <br>try restarting transaction

> 分析：
> 1. 由于数据库已经存在该记录，所以事务T1执行insert ignore into 会插入失败，并给该记录加了个S锁。
> 2. 由于S锁是相互兼容的，所以事务T2也给该记录加了S锁。
> 3. T1继续执行update语句，尝试给2行数据加X锁，但是其中有一行数据已经被T2加了S锁，此时T1回到等待队列中继续等待。
> 4. T2继续执行update语句，尝试给2行数据加X锁，但是发现T1已经对这2行数据请求了X锁，且在等待T2释放S锁，而T1又因为T2不释放S锁而无法升级为X锁。
> 
> 可以参考mysql官方的例子，原理是一样的。[innodb-死锁例子](https://dev.mysql.com/doc/refman/5.7/en/innodb-deadlock-example.html)

下面是场景二的DEADLOCK信息（show engine innodb status），你会发现其实跟生产环境的锁是又区别的，线上的死锁信息中T2 持有的是一个X锁（这个不知道怎么解释，无法重现）
```
LATEST DETECTED DEADLOCK
------------------------
2019-02-27 14:15:00 0x7000034b5000
*** (1) TRANSACTION:
TRANSACTION 620686, ACTIVE 19 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 5 lock struct(s), heap size 1136, 4 row lock(s)
MySQL thread id 63, OS thread handle 123145355350016, query id 1172740 localhost 127.0.0.1 root updating
update fans
		set following_each_other = 1
		where (fans_user_id = '11000' and following_user_id = '10086' ) or (fans_user_id = '10086' and following_user_id = '11000' )
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 107 page no 7 n bits 624 index UK_USER_ID of table `test`.`fans` trx id 620686 lock_mode X locks rec but not gap waiting
Record lock, heap no 280 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 5; hex 3131303030; asc 11000;;
 1: len 5; hex 3130303836; asc 10086;;
 2: len 8; hex 800000000143abeb; asc      C  ;;

*** (2) TRANSACTION:
TRANSACTION 620687, ACTIVE 16 sec starting index read
mysql tables in use 1, locked 1
3 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 61, OS thread handle 123145357578240, query id 1172741 localhost 127.0.0.1 root updating
update fans
		set following_each_other = 1
		where (fans_user_id = '11000' and following_user_id = '10086' ) or (fans_user_id = '10086' and following_user_id = '11000' )
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 107 page no 7 n bits 624 index UK_USER_ID of table `test`.`fans` trx id 620687 lock mode S
Record lock, heap no 280 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 5; hex 3131303030; asc 11000;;
 1: len 5; hex 3130303836; asc 10086;;
 2: len 8; hex 800000000143abeb; asc      C  ;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 107 page no 7 n bits 624 index UK_USER_ID of table `test`.`fans` trx id 620687 lock_mode X locks rec but not gap waiting
Record lock, heap no 279 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 5; hex 3130303836; asc 10086;;
 1: len 5; hex 3131303030; asc 11000;;
 2: len 8; hex 800000000143abdb; asc      C  ;;

*** WE ROLL BACK TRANSACTION (2)

```


## 解决方法
在这个案例中，发生死锁的原因主要是insert ignore 在数据已经存在时只是加了S锁。所以解决的办法其实又几个。
1. 使用其他的幂等处理办法，不要依赖insert ignore。在这个案例中，其实应该直接使用insert，不允许重复执行，可以捕获唯一key异常来获取updateNum进行下个业务处理。
2. 直接在业务开头使用select for ... update 来加排他锁保证业务是串形执行（只是从死锁这个问题考虑，如果考虑性能需要找其他方案）。
3. 假如就非得使用insert ignore 和 update，那么我们可以考虑在这个业务加个重试，数据库的死锁并不是致命的，设置好数据库的事物超时时间，然后遇到死锁问题，我们可以在业务进行重试解决。

## 未分析清楚的点
1. 为什么线上的锁他是一个X锁，并不是一个S锁？是否有场景三？MySQL官方文档有一段话，感觉有点关联，但是无法对应上现象,说的是insert 和 delete 语句其实并不是真正原子的行锁。
> InnoDB uses automatic row-level locking. You can get deadlocks even in the case of transactions that just insert or delete a single row. That is because these operations are not really “atomic”; they automatically set locks on the (possibly several) index records of the row inserted or deleted.


