---
title: MySQL的事务与锁
date: 2019-10-09 23:37:00
updated: 2019-10-09 23:37:00
tags:
  - MySQL
  - 事务
  - 锁
categories: 
  - 实践
  - 技术小结
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

### 2.1.1 共享锁和排他锁

标准的行锁实现：
S 锁：读取行的事务

# 1. 概念

表锁：
普通表锁：分 S 锁和 X 锁
意向锁：分 IS 锁和 IX 锁
自增锁：一般见不到，只有在 innodb_autoinc_lock_mode = 0 或者 Bulk inserts 时才可能有

行锁：行锁都是加在索引上的，最终都会落在聚簇索引上。加行锁的过程是一条一条记录加的；
记录锁：分 S 锁和 X 锁
间隙锁(GAP锁)：分 S 锁和 X 锁
Next-key 锁：分 S 锁和 X 锁
插入意向锁


SELECT ... 语句正常情况下为快照读，不加锁；
SELECT ... LOCK IN SHARE MODE 语句为当前读，加 S 锁；
SELECT ... FOR UPDATE 语句为当前读，加 X 锁；
常见的 DML 语句（如 INSERT、DELETE、UPDATE）为当前读，加 X 锁；
常见的 DDL 语句（如 ALTER、CREATE 等）加表级锁，且这些语句为隐式提交，不能回滚；


# 3. 参考

https://www.aneasystone.com/archives/2018/04/solving-dead-locks-four.html
https://github.com/aneasystone/mysql-deadlocks
https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html#innodb-next-key-locks