# 通过更新语句了解redo log和binlog

之前已经了解过了一条查询语句的执行过程，更新语句与查询语句流程类似，如果一个表有更新，在通过查询缓存时这条语句会将**对应表上的所有缓存结果清空**。

之后分析器通过词法分析和语法分析了解到这是一条更新语句，优化器决定索引，执行器负责执行，找到具体行然后更新。

最重要的不同在于，更新的过程中还涉及到了两个日志模块，**redo log 和 binlog**。

## redo log

如果每一次更新操作，都去刷磁盘，过程中的IO成本以及查找成本较高，因此redo log采用了 **WAL（Write-Ahead Logging）技术**，即**先写日志，待时机合适再写磁盘**。

> 写日志不也要写IO吗？
>
> 写日志确实要写IO，但是写日志是顺序IO，速度很快，而更新操作时的IO是随机IO，效率较低。

也就是当一条记录需要更新时，先将要更新的内容，写入redo log，再更新内存，这样这个更新操作就算完成了。适当的时候，InnoDB引擎会将记录中的操作再刷入磁盘里，通常这时候时系**统较为空闲时**操作。

但是redo log有大小限制（固定大小，循环写入），如果redo log满了，那么就必须**先将部分内容刷入磁盘，之后再将刷入部分擦除**，腾出新空间。

![img](https://oylong-blog-pic.oss-cn-shenzhen.aliyuncs.com/blog/img/16a7950217b3f0f4ed02db5db59562a7.png)

write pos表示当前记录的位置，写了之后往后移动，check point表示要擦除的位置，擦除后刷磁盘，同样往后移动。

write pos和 check point之间的**绿色部分表示可用部分**，**橙色部分表示已经被使用的部分**，如果write pos 追上了 check point，那么就必须擦除部分日志，将check point往前推进。

有了redo log，我们也称**InnoDB引擎拥有了 crash-safe能力**，即数据库发生异常重启，之前提交的记录不会丢失。

## binlog

redo log是InnoDB独有的，处于引擎层，而**Server层也有其独有的日志，称为binlog**（归档日志）。

redo log和 binlog有几点不同

- redo log为InnoDB引擎独有，binlog为Server层实现，所有引擎都可以使用
- redo log是物理日志，记录的是“某个数据做了什么修改”；binlog是逻辑日志，记录的是语句的原始逻辑，如“某行某个字段加了1”
- redo log固定大小，循环写入；binlog是追加写入



## 两阶段提交

![img](https://oylong-blog-pic.oss-cn-shenzhen.aliyuncs.com/blog/img/2e5bff4910ec189fe1ee6e2ecc7b4bbe.png)

> 浅色表示引擎中执行，深色表示执行器中执行

执行器在引擎中的执行流程：

1. 执行器通过引擎找到指定行，如果行所在的页本来就在内存中，直接返回给执行器，否则需要先读入内存再返回。
2. 执行器拿到引擎返回的数据，将值做更改，然后调用引擎接口写入新值
3. 将新数据更新到内存，同时写redo log，此时**redo log处于prepare状态**。随后告知执行器执行完成，可以提交。
4. 执行器生成binlog，并且将binlog写盘。
5. 执行器调用引擎提交事务接口，引擎把刚刚写入的**redolog改为提交（commit）状态**，更新完成。

其中 prepare->commit就是两阶段提交，用于保证redo log和binlog一致性

1. redo log为commit状态

   有效，将事务提交

2. redo log为prepare状态

   检查binlog，如果binlog记录成功，则有效，事务提交

   如果binlog没有记录，则事务回滚



## 两个参数

`innodb_flush_log_at_trx_commit`参数用于控制redo log的刷盘时机，默认为1

- 为0时，表示每秒将redo log buffer数据写入磁盘
- 为1时，表示每次事务提交时将redo log buffer写入磁盘文件
- 为2时，把redo log写入磁盘文件对应的os cahe，可能1s侯才会把os cache的数据写入磁盘文件



`sync_binlog`参数用于控制binlog的刷盘时机，默认为0

- 为0时，表示事务提交后写入操作系统缓存
- 为1时表示事务提交则刷入磁盘
- 为N时表示由操作系统控制刷盘时机

