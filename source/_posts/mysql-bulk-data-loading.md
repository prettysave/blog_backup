---
title: mysql中大量数据插入的优化（innodb篇）
date: 2020-09-23 10:48:03
categories: 数据库
tags:
- mysql
- 优化 
---

偶然遇到一个面试题：MYSQL插入大量数据（例如100W条）时怎么优化。进过一番查找，最后在官方文档发现了相关信息，在此做个记录.

官方文档连接:[innodb插入大量数据的优化建议](https://dev.mysql.com/doc/refman/5.6/en/optimizing-innodb-bulk-data-loading.html)

<!--more-->

## **关于事务提交的优化**

关闭事务的自动提交模式,因为innodb的事务持久性保证需要在每一个insert事务提交前，将redo log刷回磁盘。手动提交可以进行批量的插入数据，手动提交之后才将log刷回磁盘，这样可以减少磁盘IO。

`SET autocommit=0;` 

`SQL import statements`

`COMMIT;`

## **关于唯一索引的优化**

如果在二级索引列上包含唯一性检查（即唯一索引）,可以再插入数据前临时关闭唯一性检查，数据插入之后再开启。

`原理`：innodb的唯一索引，再更新时，需要实时的进行唯一性约束检查，这一步必须要刷回磁盘。而关闭了唯一性检查之后，innodb可以利用change buffer批量的刷回磁盘，从而减少了磁盘IO。

`tips`：**这个操作有一个前提就是插入的数据必须保证不能有唯一性冲突**

`SET unique_checks=0;`
`SQL import statements`
`SET unique_checks=1;`

## **关于外键的优化**

如果表中含有外键，可以暂时关闭约外键约束检查，语句执行之后再开启。这一步同样是可以减少磁盘IO；（外键并没有用过，这里只是照搬官方文档的说法）

`SET foreign_key_checks=0`

​	 `SQL import statements` 

`SET foreign_key_checks=1;`

## **关于Insert语句的优化**

使用多条插入的语法来减少客户端和服务器之间的通信成本

Tips:这一条适用于任何的表，而不仅仅是Innodb的表.

`INSERT INTO yourtable VALUES (1,2), (5,5), ...;`

## **关于自增锁模式的优化**

将innodb_antoinc_lock_mode改为2（默认是1）,好处是不会使用table-level lock这样可以并发的执行SQL语句，缺点是同一时刻多条SQL语句产生交错的auto-increment值，在使用SBR复制或者回复场景中回放binary log是不安全的（我的理解是无法保证每次执行获得的auto_increment值都是一样的。

## **关于主键的优化**

按照主键的顺序排序插入，一般使用innodb的时候都会声明一个自增的主键（没有显示声明，则mysql会自动选择一个具有唯一性约束的索引作为主键，如果没有唯一索引，则会隐式的创建一个自增主键），innodb使用聚簇索引保存数据，即数据会按照主键顺序的进行存储，按照主键顺序插入的话避免的页分页等问题，也可以完美的使用innodb中的buffer pool技术，减少了磁盘IO。

## **关于全文索引的优化**

emmm，全文索引不了解，暂时略过



**上面介绍的是innodb的大数据量插入优化，官方文档中还有MyISAM的优化建议，因为本人对MYISAM了解甚少，暂时不做总结，有兴趣自行前往查看**：

[ MyISAM插入大量数据的优化建议](https://dev.mysql.com/doc/refman/5.7/en/optimizing-myisam-bulk-data-loading.html)

