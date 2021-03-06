# 一致性读与当前读

开启一个事务，可以通过`begin`或`start transaction`命令开启，但是这两个命令并不是一个事务的起点，**事务真正开启是在执行第一个操作InnoDB表的语句**之后，才是真正启动。这样可以最大程度上支持事务并发，让锁持有的时间尽量短。

如果想要立马启动一个事务，那么可以通过`start transaction with consistent snapshot`命令去开启事务。



## 快照

可重复读隔离级别下，事务启动之后会拍快照，且是基于整库的快照。

快照的实现是通过事务的唯一ID，**transactional id**实现的，每次事务更新数据时，都会生成一个新的数据版本，并且把**transactional id**赋值给新数据的事务ID，称为**row trx_id**。同时旧的数据版本也会保留。如下图：

![img](https://oylong-blog-pic.oss-cn-shenzhen.aliyuncs.com/blog/img/68d08d277a6f7926a41cc5541d3dfced.png)

这里也引出了我们MySQL中的**undo log**，我们版本的回退链就构成了我们的undo log。

在可重复读的隔离级别下，事务启动后，其他事物更新对当前事务不可见。意思就是如果当前版本数据的事务id大于我的事务id，那么我就不认，往前去寻找，如果版本数据的事务id小于或者为自己的事务id，那么便可见。



## 一致性读

实现上，InnoDB为每个事务都构造了一个数组，用来保存事务启动瞬间，当前依然存在的事务的所有事务ID。

事务ID的**最小值称为低水位，对大值+1称为高水位**。视图数组和高水位就构成了我们的**一致性视图**。

如下图，我们可以把事务启动瞬间的视图数组分为以下几种情况

![img](https://oylong-blog-pic.oss-cn-shenzhen.aliyuncs.com/blog/img/882114aaf55861832b4270d44507695e.png)

- 低于低水位的，表示这些版本为已经提交的事务或者是当前事务自己生成的，可见。

- 高水位之后的，表示将来启动的事务，肯定不可见

- 中间的，表示未提交的事务，可分为两种情况。

  1. 如果`row trx_id`在数组中，表示事务还没有提交，不可见。

  2. 如果`row trx_id`不在数组中，那么说明十五已经提交了，可见。

     > 注：数组里面的都是活跃的数组，就是还没有提交的事务。

  InnoDB就是通过这样的**一个数据有多个版本**的特性，实现了*秒级创建快照*的能力。

因此，如果一个事务启动了，那么在之后无论别的事物怎么修改，当前事务的查询结果都不会变，这也称为**一致性读**。

> 前提是当前事务没有更新过



## 当前读

而更新逻辑则不同，**更新数据都是先读后写的，而这个读则会读版本中的最新值，称为当前读**。除了`update`语句外， `select`语句如果加了锁，则也是当前读。

**当前读会读最新的版本，如果最新版本的事务未提交，由于行锁的原因，语句会阻塞，必须等待最新版本的事务提交或回滚**。



