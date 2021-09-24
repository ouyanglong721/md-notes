# MySQL-03 了解MySQL事务

>  事务就是保证一组数据库操作，要么全部成功，要么全部失败。
>
> MySQL中，事务支持实在引擎层中实现的，不是所有的引擎都支持事务，如*MyISAM*引擎就不支持，而取代它的InnoDB则支持事务特性。

## 事务的隔离性与隔离级别

隔离性也就是我们常说的ACID中的I（Isolation）

多个事务同时进行时，可能出现脏读，不可重复读，幻读的问题，为了解决这些问题，出现了“隔离级别”的概念。

SQL标准的事务隔离级别包括：

- 读未提交

  一个事务没提交，它做的变更别的事务就能看到。

- 读提交

  一个事务提交之后，它做的变更别的事物才能看到。

- 可重复读

  一个事务执行过程中看到的数据，总是跟这个是事务启动时看到的数据是一只的。

- 串行化

  读写会加锁，出现冲突时，后访问的被阻塞

**MySQL默认的隔离级别为可重复读**。

**MySQL实现不同的隔离级别主要通过一个视图区分，读未提交下，不创建视图；读提交下，在每个SQL语句开始执行时创建；可重复读下，在事务启动时创建；创行话通过加锁的方式避免并行访问。**



## 隔离的实现

MySQL中，每条记录在更新时，**都会通过链表记录一条回滚操作**（undo log），记录的最新之通过这个链表进行回滚，能得到之前的旧值。

![img](https://oylong-blog-pic.oss-cn-shenzhen.aliyuncs.com/blog/img/d9c313809e5ac148fc39feff532f0fee.png)

在不同的时刻启动事务会得到不同的read-view，同一条记录在系统中能存在多个版本，这就是数据库的多版本并发控制（MVCC）。

日志将在不需要使用的时候被删除，即系统里面没有比这个read-view更早的事务时。

因此不建议使用长事务，长事务会导致系统里存在老试图，占用大量存储空间。



## 启动事务

1. `begin`或`start transcation`配合`commit`和`rollback`使用。

2. `set autocommit=0`命令关闭自动提交，每执行一个select语句，事务就启动了，且不会自动提交，手动`commit`或`rollback。`

3. `set autocommit=1`自动提交。

在 autocommit 为 1 的情况下，用 `begin` 显式启动的事务，如果执行 `commit` 则提交事务。如果执行 `commit work and chain`，则是提交事务并自动启动下一个事务，这样也省去了再次执行 begin 语句的开销。同时带来的好处是从程序开发的角度明确地知道每个语句是否处于事务中。

在`information_schema`库中的`innodb_trx`可以查询当前事务。

下面的SQL语句可用于查询超过60s的事务

`select * from information_schema.innodb_trx where TIME_TO_SEC(timediff(now(),trx_started))>60`

   