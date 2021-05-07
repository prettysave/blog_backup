---
title: SELECT ... FOR UPDATE
date: 2020-11-25 17:10:20
categories: 数据库
tags: 
- mysql
- 锁 
---

## 解决的问题

1.并发更新冲突，保证只有一个事务能更新

2.取出最新的数据

## 是什么

首先是一种悲观锁，还有**SELECT … LOCK IN SHARE MODE**同为悲观锁。其次，是一种独占锁。而**SELECT … LOCK IN SHARE MODE**则为共享锁。

### 悲观锁和乐观锁的区别:

- 悲观锁

正如其名，它指的是对数据被外界（包括本系统当前的其他事务，以及来自外部系统的事务处理）修改持保守态度，因此，在整个数据处理过程中，将数据处于锁定状态。悲观锁的实现，往往依靠数据库提供的锁机制（也只有数据库层提供的锁机制才能真正保证数据访问的排他性，否则，即使在本系统中实现了加锁机制，也无法保证外部系统不会修改数据）。

在悲观锁的情况下，为了保证事务的隔离性，就需要一致性锁定读。读取数据时给加锁，其它事务无法修改这些数据。修改删除数据时也要加锁，其它事务无法读取这些数据。

- 乐观锁

相对悲观锁而言，乐观锁机制采取了更加宽松的加锁机制。悲观锁大多数情况下依靠数据库的锁机制实现，以保证操作最大程度的独占性。但随之而来的就是数据库性能的大量开销，特别是对长事务而言，这样的开销往往无法承受。

而乐观锁机制在一定程度上解决了这个问题。乐观锁，大多是基于数据版本（ Version ）记录机制实现。何谓数据版本？即为数据增加一个版本标识，在基于数据库表的版本解决方案中，一般是通过为数据库表增加一个 “version” 字段来实现。读取出数据时，将此版本号一同读出，之后更新时，对此版本号加一。此时，将提交数据的版本数据与数据库表对应记录的当前版本信息进行比对，如果提交的数据版本号大于数据库表当前版本号，则予以更新，否则认为是过期数据。

– [引自美团技术团队博客](https://tech.meituan.com/2014/08/20/innodb-lock.html)

## 原理

要了解其背后的原理需要先了解一下mysql关于事务相关的一些知识：

### ACID-事务基本要素

 A（atomicity）原子性:一个事务中的所有操作，要么全部完成，要么全部不完成
​ C（consistency）一致性:在事务开始之前和事务结束之后，数据库的完整性约束没有被破坏。
​ I（isolation）隔离性：一个事务的执行不能影响到其它事务的执行，并发执行的各个事务之间不能互相干扰。
​ D（durability）持久性：事务完成以后，该事务对数据库所做的更改便持久的保存在数据库中。

### 并发事务处理的问题

- 更新丢失：事务A和事务B选择同一行，然后基于最初选定的值更新该行时，由于两个事务都不知道彼此的存在，就会发生丢失更新问题
- 脏读：事务A读取了事务B未提交的数据，然后事务B回滚,A读到的数据就是脏数据,
- 不可重复读：事务A多次读取同一数据，事务B在事务A多次读取的过程中，对数据做了更改并提交，导致事务A多次读取同一数据时，结果不一致。
- 幻读：类似于不可重复读，不可重复读指的是同一数据多次读取不一致，幻读是指在一个范围内，多次读取的数据条数不一致.

### 事务隔离级别

- 读未提交（read uncommitted）：最低的隔离级别，可能导致脏读、幻读、不可重复读
- 读已提交（read committed）：允许读取并发事务已经提交的数据，可以阻止脏读，可能发送幻读、不可重复读
- 可重复读（repeatable read）：可以阻止脏读和不可重复读，但幻读仍有可能发生。(innodb中通过mvcc避免的幻读)
- 可串行（serializable）：最高的隔离级别，所有事务依次逐个执行，虽然可以防止所有问题。但是基本上不用

查看数据库的隔离级别：show variables like ‘%tx_isolation%’ ;

mysql默认的隔离级别是repeatable read；

### MVCC多版本并发控制(innodb)

committed read | repeatable read两种隔离级别下工作

#### 实现

 聚簇索引列中额外记录了两个必要的隐藏列：
​ trx_id：记录改动后最新的事务ID
​ roll_pointer:相当于指针，可以通过它在undo日志中找到上一个版本的数据
read view:包含当前系统中的活跃的读写事务的id列表,命名为m_ids
​ 如果被访问版本的trx_id小于m_ids中的最小值，说明生成该版本的事务在生成read view的时候已经提交，改版本可访问
​ 如果trx_id大于m_ids中的最大值，表明生成该版本的事务在生成read view后才生成，不可访问
​ 如果trx_id在mids的最大值和最小值之间，则判断是否在mids列表中，存在说明创建read view的时候该版本的trx_id的还未提交，不可访问；如果不在，可访问。
​ read committed – 每次读取数据都生成一个read view
​ repeatable read – 只在第一次读取数据时生成一个read view
​ 在Innodb的mvcc实现下，repeatable read的实现是可以解决幻读问题的。

**快照读**：读取的数据是在当前事务开始时，其他事务已经提交的数据。普通的查询都是快照读。

**当前读**：读取的数据可以是当前事务进行中其他事务提交的数据

## 缺点

1.多个事务竞争时会导致其它事务阻塞，会影响并发性能，所以要尽量避免大事务。

2.innodb的锁在执行计划走索引时会给索引列上行锁，但是当执行计划解析之后没有索引项可锁就会锁表，这个时候对并发的影响是巨大的，此时其他事务只要是针对这个表的上锁都会被阻塞，所以一定要保证能够走索引。

– 有时候哪怕是sql语句中的where条件创建的索引，最终的执行计划也不一定会走索引的，这是因为当mysql的优化器觉得全表扫描会比走索引快时就会放弃索引，直接进行全表扫描。

3.产生死锁。一个事务先锁A，再锁B；一个事务先锁B，后锁A。

–为了避免这种情况，业务代码中尽量使用一致的上锁顺序。