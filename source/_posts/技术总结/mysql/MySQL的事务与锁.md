---
title: MySQL的事务与锁
date: 2019-10-09 23:37:00
updated: 2019-10-09 23:37:00
tags:
  - mysql
  - 事务
categories: 
  - [技术总结], mysql]
comments: true
permalink: tech/mysql_transction_lock.html    
---

# 0 背景

做支付碰到的死锁解决起来有点摸不着头脑，因此重新学了一遍 MySQL 中的事务与锁的知识，主要是基于 InnoDB

<!--more-->

---

# 1. 事务

事务需要保证 ACID，而在MySQL 中使用了：日志文件、锁、MVCC 来实现事务

## 1.1 日志文件

### 1.1.1 redo log

redo log：即重做日志，用来记录已成功提交事务的修改信息，并且会持久化到磁盘。用来实现事务的持久性，由两部分组成：redo log buffer 以及 redo log，前者在内存中，后者在磁盘中。  
举例来说明 redo log 里面存储的数据：
```sql
insert into bank(name, balance) values ('zhangsan', 1000);
insert into finance(name, amount) values ('zhangsan', 0);
start transaction;
// 生成 redo log：balance=600
update bank set balance = balance - 400;
// 生成 redo log：amount=400
update finance set amount = amount + 400;
commit;
```

mysql 为了提高性能，在修改操作时，不会实时的同步到磁盘，而是先放入缓存池中，然后后台线程做缓冲池和磁盘之间的同步，为了防止在同步时，机器宕机，导致丢失部分已提交事务的修改信息，引入了 redo log。  
而 redo log 是持久化在磁盘中的。   
redo log 是用来恢复数据的，用于保障已提交事务的持久化特性

### 1.1.2 undo log

undo log：即回滚日志，用于记录数据被修改前的信息。  
举例来说明 undo log 里面存储的数据：
```sql
insert into bank(name, balance) values ('zhangsan', 1000);
insert into finance(name, amount) values ('zhangsan', 0);
start transaction;
// 生成 undo log：balance=1000
update bank set balance = balance - 400;
// 生成 undo log：amount=0
update finance set amount = amount + 400;
commit;
```

undo log 用于数据回滚，保证未提交事务的原子性。注意：undo log 也会产生 redo log 文件来保证 undo log 的持久性。

## 1.2 MVCC

MVCC 基于行的三个隐含字段：rowid（如果没有主键才有该字段作为隐藏主键）、事务号、回滚指针  
具体没有深入，不知道

# 2. 锁

只考虑 MySQL 的 RR 隔离级别下的 InnoDB 引擎的锁

## 2.1 锁的种类

来自官网：https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html

Shared and Exclusive Locks

Intention Locks

Record Locks

Gap Locks

Next-Key Locks

Insert Intention Locks

AUTO-INC Locks

Predicate Locks for Spatial Indexes

注意行锁：
>行锁都是加在索引上的，最终都会落在聚簇索引上。加行锁的过程是一条一条记录加的

### 2.1.1 Shared and Exclusive Locks

又称共享锁和独占锁，作为标准的行锁实现  
共享锁也叫 S 锁，允许持有锁读取行的事务
独占锁也叫 X 锁，允许持有锁更新或删除行的事务
 
### 2.1.2 Intention Locks

意向锁，作为表锁实现，有两种：意向共享锁，意向独占锁。
一般在设置某行的共享锁或独占锁时，系统会自动给该表添加意向共享锁或意向独占锁： 
意向共享锁（IS）：表示一个事务在对该表某行进行读取行操作时，会自动申请该表的 IS 锁。
意向独占锁（IX）：表示一个事务在对该表某行进行更新或删除行操作时，会自动申请该表的 IX 锁。

为什么会有 IS 和 IX，是因为当需要对表做读取或写入操作时，不用遍历每行数据，来判断是否有行已经被持有了锁。  

引用知乎：

>考虑这个例子：
>事务A锁住了表中的一行，让这一行只能读，不能写。
>之后，事务B申请整个表的写锁。
>如果事务B申请成功，那么理论上它就能修改表中的任意一行，这与A持有的行锁是冲突的。
>数据库需要避免这种冲突，就是说要让B的申请被阻塞，直到A释放了行锁。
>数据库要怎么判断这个冲突呢？
>step1：判断表是否已被其他事务用表锁锁表
>step2：判断表中的每一行是否已被行锁锁住。
>注意step2，这样的判断方法效率实在不高，因为需要遍历整个表。于是就有了意向锁。
>在意向锁存在的情况下，事务A必须先申请表的意向共享锁，成功后再申请一行的行锁。
>在意向锁存在的情况下，上面的判断可以改成
>step1：不变
>step2：发现表上有意向共享锁，说明表中有些行被共享行锁锁住了，因此，事务B申请表的写锁会被阻塞。
>注意：申请意向锁的动作是数据库完成的，就是说，事务A申请一行的行锁的时候，数据库会自动先开始申请表的意向锁，不需要我们程序员使用代码来申请。
>
>作者：发条地精
链接：https://www.zhihu.com/question/51513268/answer/127777478
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

### 2.1.3 Record Locks

记录锁：对索引记录的锁定，行锁实现

记录锁定始终锁定索引记录，即使没有定义索引的表也是如此。对于这种情况，InnoDB 会创建一个隐藏的聚集索引，并将该索引用于记录锁定

### 2.1.4 Gap Locks

间隙锁：对索引记录之间的间隙的锁定，例如：
```sql
-- id 为主键，并且表中已经有了 id = 50, id = 150 的记录
-- 那么在事务中，作为间隙锁下面语句将会锁定 id(50, 100), id(100,150) 的间隙，阻塞这两个间隙的记录插入
SELECT * FROM child WHERE id = 100;
```

### 2.1.5 Next-Key Locks

Next-Key Locks：是记录锁和间隙锁的组合，因为间隙锁的范围会开区间，而 Next-Key Locks 结合了记录锁，会形成闭区间：  
(50, 100], (100, 150] 的 Next-Key Locks

### 2.1.6 Insert Intention Locks

插入意向锁：通过 INSERT 行插入之间的操作设置间隙：
```sql
-- 表中有 id=90, id=102 两条数据
START TRANSACTION;
SELECT * FROM child WHERE id > 100 FOR UPDATE;
-- 执行 TransactionB，以下语句会被阻塞
INSERT INTO child (id) VALUES (101);
```

### 2.1.7 AUTO-INC Locks

自动上锁：表中一个特殊的表级锁 AUTO_INCREMENT列，即保证主键插入的自增

### 2.1.8 Predicate Locks for Spatial Indexes

不知道

# 3. 死锁日志分析

强推大佬的：https://github.com/aneasystone/mysql-deadlocks.git 收集了基本上所有的死锁案例并复现了基本所有案例。线上排查死锁必备。  
为什么写这篇博文，也是因为该项目如果没有一定的事务和锁基础很难看懂  
我是先看了 《MySQL 技术内幕》 一书然后再看该项目作者关于死锁博文系列：https://www.aneasystone.com/archives/2018/04/solving-dead-locks-four.html  
最后给自己总结的该博文，可能会有误导，因为自己知识有限。