﻿# InnoDB存储引擎的并发控制


---


InnoDB 是 MySQL 默认的存储引擎；与 MyISAM 相比，InnoDB 则支持多用户同时操作的事务处理和多粒度锁控制。

> 事务是并发控制的基本单位。保证事务的 ACID 的特性是事务处理的重要任务，为了保证事务的隔离性和一致性，需要对并发操作进行正确的调度。 --《数据库系统概论》

针对 MySQL 数据库，粒度锁和事务处理是在存储引擎层面上实现的。 InnoDB 支持 SQL 标准制定的四种事务隔离等级，四种事务级别分别是：
`REPERATABLE READ`, `READ UNCOMMITED`, `READ COMMITED`, `SERIALIZABLE`,其中 `REPERATABLE READ` 为 MySQL 默认事务等级。
在事务处理中， MySQL 是实现两段锁（two-phase locking）处理，在事务开始是获取锁，在事务提交或发生回滚时释放获得的锁，实现并发调度。

## 锁的类型

InnoDB 具有多封锁粒度，它实现了行级锁，使得多用户可以同时事务处理。与此同时，为了实现多粒度锁，在行级锁之上又实现的了表级的意向锁。

### 共享锁和排它锁

`共享锁（S锁，Shared locks）`又称为读锁，允许事务有读的权限。
`排它锁（X锁，Exclusive locks）`又称为写锁，允许事务有 `UPDATE`，`DELETE`的权限。
MySQL两种锁模式策略与线程控制中的读写锁策略类似。S 锁和 X 锁的锁定规则如下：

1. 如果事务 `T1` 对记录 `R` 行上有 S 锁，其它事务 `T2` 请求 `R` 上的 `S`锁，可以授权，而 X 锁不能授权。 
2. 如果事务 `T1` 对记录 `R` 行上有 X 锁，则阻塞其他事务的任何请求，只有当 `T1` 事务提交或者回滚释放锁后，才能处理请求。

简单的讲，多个事务能够同时读，但不能同时写。

### 意向锁（Intention Locks）

InnoDB 中意向锁是表级锁，意向锁的意义在于说明当前存在事务正在进行或者将要对表中的记录加上某种类型的锁。
意向锁有以下三种类型：
`意向读锁（IS 锁，Intention Shared Locks）`：事务 T 将会在 t 表中的记录 R 上设置 S 锁;
`意向写锁（IX 锁，Intention Exclusive Locks）`：事务 T 将会在 t 表中的记录 R 上设置 X 锁;
`插入意向锁（Insert Intention Locks）`: 用于 InnoDB 的 Gap Locks 中。

意向锁的策略：

1. 事务在表中记录行上有 S 锁，必须先获得该表的 IS 锁或者等级更高的锁；

2. 事务在表中记录行上有 X 锁，必须先获得该表的 IX 锁。

锁兼容性表：

|\   |X   |IX  |S   |IS  |
|:--:|:--:|:--:|:--:|:--:|
|X   |N   |N   |N   |N   |
|IX  |N   |Y   |N   |Y   |
|S   |N   |N   |Y   |Y   |
|IS  |N   |Y   |Y   |Y   |

从兼容性表可以了解锁的等级存在一个排序关系， X 锁的等级最高， IS 锁的等级最低。

---

## 读取控制

在 InnoDB 中存在两种读方式，一种是不加锁实现读的一致性，另一种是通过加入行锁实现读取数据的一致性。

### 一致性读（Consistent Reads）

一致性读是通过 InnoDB 特有的存储方式实现，在默认事务 `REPERATABLE READ` 和 `READ COMMITED` 级别中，这种读方式应用于普通的 `SELECT` SQL 语句。一致性读读取的是 InnDB 显示某一个时间节点的快照，所以不会在数据表中加任何锁，也不会对其他事务造成阻塞，与此同又时保证了事务之间一定程度的隔离性。

一致性读又称为快照读。在 `REPERATABLE READ` 事务中，每次读取的是当前事务建立的快照。 在 `READ COMMITED` 事务级别中，一致性读的快照每次都是最新的，因为在这个级别中每次读完后都会提交事务。

一致性读依赖于 InnoDB 存取表记录的特殊结构以及 `Update undo log`。
InnoDB 在存取数据表记录时会在每一行添加三个字段。其中一个字段就是插入或者更新当前行的最新事务 ID。另外 InnoDB 会根据 `Update undo log` 建立快照。所以如果 undo log 未记录的， consistent reads 就无法使用， 所以对于 `DROP TABLE` 和 ` ALTER TABLE` 的记录是无效的。

### 锁定读（Locking Reads）

因为一致性读对 SELECT 查询的结果安全性不足，存在其他事务修改查询的结果，这样就会出现读‘脏’数据的可能， InnoDB 利用封锁机制，提供了两种类型的锁定读。

1. `SELECT ... LOCK IN SHARE MODE` 这是在选取的行上设置了一个 S 锁。在设置 S 锁之前，需要获取一个 IS 的表锁，通过这种锁机制，使得其他事务只能读，而不能写这些记录，如 S 锁的规则。
2. `SELECT ... FOR　UPDATE` 这个类型的锁就是在选取的记录行上添加 X 锁，在这之前必须先要获得表级的 IX 锁。这个锁会阻塞其他所有需要改记录的事务，直到这个锁释放。

这两种模式的锁是设置在 B+ Tree 的索引记录上。因为 一致性读不依赖于树结构而是通过 undo log 读快照实现的，所以一致性读仍然可以读到数据，因此它会忽略这些锁的限制。

---

## InnoDB 中的锁

InnoDB的是多粒度的封锁机制，最细粒度的锁是行级锁，InnoDB 实现了三种类型的行级锁。

### Record Locks

记录锁（Record Lock）是在索引记录上建立的锁机制，它能锁住单条记录而不影响其他事务对其他记录的读写，记录锁也分为读模式和写模式。
另外一点是关于 InnoDB 中索引的问题，一般来讲，InnoDB 实现的是聚簇索引，索引叶节点直接连接着记录行。这不像 MyISAM 引擎，MyISAM 实现的是非聚簇索引，它的索引叶节点指向记录的指针。基于上种情况，在使用 InnoDB 时应该尽可能的声明主键，如果没有声明主键，InnoDB 会根据 unique 字段建立索引，如果 unique 字段也没有。InnoDB 在记录行添加的三个字段中其中一个就是行标志符，可根据此字段建立索引。而 MyISAM 则不需要主键。

### Gap Locks

Gap Locks 中文名不详，这个锁 的功能是可以锁住多个行，是锁住两个索引记录之间的记录行。一个间隔可能会锁住一个索引或者是多个索引，也会可以为空。

有一种 gap lock 称为插入意向锁，而这个锁是针对插入操作的，它表明的是只要多个事务不是插入同一个位置，这些事务可以不用等待就可以在这个间隔中直接插入记录。

### Next-key Locking

Next-key Lock 又称为间隙锁，这个锁是 record lock 和 gap lock 的组合，指的是锁住一个记录的同时，锁住这条记录前面的一段索引。
间隙锁的主要作用是避免幻读问题。在同一事务的两次 SELECT 操作中，另一事务可能会在这些记录中添加行，从而造成两次的 SELECT 返回的结果不同，此所谓幻读。InnoDB 用 Next-key Lock 解决幻读问题。 InnoDB 利用的是插入意向锁和记录锁的组合，如果当前记录前的 gap 中有插入意向锁且当前记录有锁，这个插入意向锁会阻塞其他事务在这个 gap 中插入记录，避免幻读。

---

## 死锁（Deadlock）

DBMS 中检测死锁的方法有`超时法`和`事务等待图法`。
InnoDB 可以自动的检测死锁，并且会选择影响最小的事务回滚。InnoDB也会记录下死锁的信息，便于分析。

---

## 参考书籍

1. 《MySQL 参考手册》
2. 《高性能 MySQL》
3. 《数据库系统概论》

---
