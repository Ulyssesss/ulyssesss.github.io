---
title: MySQL 体系结构与存储引擎
date: 2018-10-18 15:29:57
tags:
  - MySQL
categories:
  - 技术
---

了解 MySQL 的体系结构可以帮助我们更好的学习 MySQL，本篇文章对 MySQL 体系结构进行了简要介绍，同时对 InnoDB 存储引擎的工作方式进行了说明。





<!-- more -->

## MySQL 体系结构

MySQL 的体系结构可以分为两层：MySQL Server 层和存储引擎层。MySQL Server 层又包括连接层和 SQL 层。

应用程序通过接口连接 MySQL，最先到达连接层，经过通信协议处理、线程处理和用户名密码认证三个部分。

SQL 层包括权限判断、查询缓存、解析器、预处理、查询优化器、缓存和执行计划。

其中查询缓存通过 Query Cache 进行操作，解析器判断语法是否正确，预处理器对解析器无法解析的语义进行处理，优化器对 SQL 进行改写优化，生成最优的执行计划，然后调用 API，通过存储引擎层访问数据。



## Query Cache

一个 SQL 如果以 select 开头，那么 MySQL 将尝试对其使用 Query Cache 。每个 Cache 都以 SQL 文本作为 key 来存储。

Query Cache 只能缓存静态数据，如果一个表被更新，和这个表相关 SQL 的所有 Query Cache 都会失效。

Query Cache 在 5.6 之前的版本中默认是开启的，5.6 之后默认是关闭的。



## 存储引擎

MySQL 及其分支版本主要的存储引擎包括：InnoDB、MyISAM、Memory、Blackhole、TokuDB 和 MariaDB columnstore。

InnoDB 支持事务、行锁，支持 MVCC（Multi-Version Concurrency Control）多版本并发控制，并发性高。

MyISAM 不支持事务，支持表锁，低并发，资源利用率也很低，在 MySQL 8.0 被废弃。

Memory 将表中数据都存在内存中，支持 Hash 和 Btree 索引，安全性不高，读取速度快。

TokuDB 支持事务、压缩功能、高速写入功能、Online DDL，不产生索引碎片。

MariaDB columnstore 列式存储引擎，支持高压缩功能。

Blackhole 并不存储数据，数据写入时只写 binlog，仓用来做 binlog 转储或测试。



## InnoDB

### 存储结构

InnoDB 逻辑存储单元分为表空间 tablespace、段 segment、区 extent 和页 page。



#### 表空间

InnoDB 引擎下表中的所有数据都存储在表空间中，表空间分为系统表空间和独立表空间。

系统表空间以 ibdata1 命名，存储所有的数据及回滚（undo）信息，数据库初始化时系统就创建了一个系统表空间文件。MySQL 5.6 后 undo 表空间可以通过参数设置将存储位置独立出来。

独立表空间即每个表有自己的表空间文件，存储对应表的 B+树数据、索引和插入缓冲等信息，其余信息仍然存储在系统表空间中。

独立表空间在效率、性能方面会由于系统表空间，MySQL 默认使用的就是独立表空间。

> MySQL 5.7 新增了临时表空间和通用表空间。
>
> 临时表空间从系统表空间中抽离，形成独立表空间；通用表空间将多个表放在同一个表空间，根据活跃度来划分表，存在不同的磁盘上，目的是减少 metadata 存储开销，生产环境很少使用。



#### 段

表空间由段组成，通常包括数据段、回滚段、索引段等，每个段由 N 个区和 32 个零散的页组成，以区为单位进行扩展。



#### 区

区由连续的 64 个页组成，是物理上连续的空间，每个区固定 1 MB（64 * 16 KB）。



#### 页

InnoDB 最小的物理存储分配单位，有数据页和回滚页等，默认为 16 KB。

MySQL 5.6 开始允许自定义调低 page 大小，可以调整为 8 KB 或 4 KB；MySQL 5.7 开始允许调高 page 大小，可以调整到 32 KB 或 64 KB。

一般情况下一个 page 页会预留 1/16 的空间用于更新数据，真正使用 15/16 的空间。



#### 行

InnoDB 引擎下数据按行存储，行按照一定的格式记录数据。

InnoDB 包含两种行存储文件格式，其中 Antelope 文件格式下有 compact 和 redundant 两种行记录格式，Barracuda 文件格式下有 compressed 和 dynamic 两种行记录格式。

目前使用最多的是 compact 行记录格式，MySQL 5.7 默认使用 dynamic 行记录格式，针对溢出列所在的新页利用率更高。

而 redundant 是最早的行记录格式，比 compact 消耗更多的存储空间；compressed 对数据和索引页进行压缩，但只在物理层面压缩，带来负面影响也很大。



### 内存结构

MySQL 内存组成分为 SGA 系统全局区和 PGA 程序全局区。



#### SGA 系统全局区

系统全局区包含以下内存区域：

innodb_buffer_pool，缓存 InnoDB 表的数据、索引、插入缓存、数据字典等信息。

innodb_log_buffer，事务在内存中的缓冲。

> Query Cache，查询缓存，建议关闭。
>
> key_buffer，只用于 MyISAM 表的索引。
>
> innodb_additional_mem_pool，保存数据字典和其他内部数据结构的内存池，MySQL 5.7.4 中被移除。



#### PGA 程序全局区

程序全局区包含以下内存区域：

sort_buffer，用于 SQL 语句在内存中临时排序。

join_buffer，表连接使用，用于 BKA （Batched Key Access  提高表 join 性能的算法）。

read_rnd_buffer，MySQL 随机读缓冲，用于做 mmr（Multi-Range Read，MySQL 5.6 优化器的新特性）。

> read_buffer，表顺序扫描的缓存，只用于 MyISAM。
>
> tmp_table SQL语句在排序或分组时没有用到索引时使用的临时表空间。
>
> max_heap_table，管理 heap、memory 存储引擎表。



### Buffer 状态

InnoDB 最小的磁盘 IO 单位是 page，对应到内存的 buffer。

buffer 分为 free、clean、dirty 三种状态，free 为从未被使用，clean 为数据和磁盘 page 一致，dirty 为数据和磁盘 page 不一致（待刷新到磁盘）。

buffer 在内存中通过 chain 链来管理，三种状态衍生出 3 条链表，free list、lru list 和 flush list。lru list 将 clean buffer 按最近最少使用串联起来，flush list 串联 dirty buffer，也隐藏 lru 规则。

数据库运行时，首先判断 free list 的使用情况，如果不够用，会从 lru 和 flush list 中释放 free buffer。



### 刷新线程

InnoDB 为多线程模型，后台有多种线程负责不同的任务。



#### master thread

主线程，优先级最高，内部包含 4 个循环：主循环 loop、后台循环 background loop、刷新循环 flush loop 和暂停循环 suspend loop。各循环根据数据状态进行切换。



#### IO thread

IO 线程，包括 redo log thread、change buffer thread 和 read / write thread。

redo log thread 负责将日志缓冲内容刷新到 redo log 文件，change buffer thread 负责将插入缓存内容刷新到磁盘，read / write thread 负责读写请求（默认 4 个，使用高转速磁盘可适当增大）。



#### page cleaner thread

负责脏页刷新，MySQL 5.7 后可增加多个。



#### purge thread

负责删除无用的 undo 页。



#### checkpoint thread

redo log 切换或文件快写满时，执行 checkpoint，触发脏页刷新到磁盘，并确保 redo log 刷新到磁盘，避免数据丢失。

>另外 error monitor thread 负责数据库报错监控，lock monitor thread 负责监控锁。



### 内存刷新机制

MySQL 和 Oracle 这种关系型数据库中，通常会先写日志再写数据文件。以下通过 redo log buffer 和 binlog cache 来介绍内存中的数据刷新到磁盘的机制。



#### redo log

redo log 又称重做日志文件，用于记录事务操作的变化，记录的是数据修改之后的值，且不管事务是否提交都会记录。故障后通过 redo log 可以恢复到之前的状态，保证数据的完整性。

默认情况 redo log 至少有两个文件，在磁盘上以 ib_logfile{n} 命名（0 ~ n）。redo log 顺序写、循环写，在第一个文件写满后，执行 checkpoint，触发脏页刷新，然后写第二个文件，全部写满后从第一个文件开始重新写。MySQL 启动时会检查参数中 redo log 的大小，如果与当前 redo log 不一致，会删除 redo log，并按照配置重新生成文件。

redo log 数据先写在 redo log buffer 中，刷新到磁盘的策略通过 innodb_flush_log_at_trx_commit 来配置。

此配置值为 0 时，redo log thread 每隔 1 秒将 buffer 中的数据写入文件，同时刷盘保证写入成功。提交事务不会触发文件的写入。配置为 1 时，每次提交事务都会写入文件并刷盘，是最安全的模式，但性能最差。配置为 2 时，每次提交事务都会写入文件，但不会同时 flush 刷盘。



#### binlog

binlog 即二进制日志文件，用于备份恢复和主从复制，通过 sync_binlog 参数来确定刷新策略。sync_binlog 的值为 0 时，有事务提交不会发出同步指令，而是让文件系统自行决定同步的时间，或 cache 满了之后才同步。sync_binlog 值为 n 时，每进行 n 次事务提交，MySQL 发出一次同步指令来将 cache 中的数据写入磁盘。

将 sync_binlog 和 innodb_flush_log_at_trx_commit 的值全配置为 1，即【双一模式】，可以最大限度保证数据库的安全性。



>redo log 和 binlog 的区别：
>
>* 记录内容不同，binlog 为逻辑日志，记录所有数据的改变信息，redo log 是物理日志，记录数据修改之后的值。
>* 记录时间不同，binlog 在 commit 之后记录，redo log 在事务发起后记录。
>* 文件使用方式不同，binlog 不循环使用，写满后生成新文件，redo log 循环使用。
>* 作用不同，binlog 用于恢复数据、主从复制，redo log 用于异常宕机后的数据恢复。



### InnoDB 三大特性

插入缓冲 change buffer、两次写 double write 和自适应哈希索引 adaptive hash index 构成 InnoDB 的三大特性，这些特性让 InnoDB 有了更好的性能和可靠性。



#### change buffer

IO 对数据库性能的影响最大，change buffer 将 DML 操作从随机 IO 变成了顺序 IO，提高 IO 效率。change buffer 先判断 DML 的普通索引页是否在缓冲池中，如果在就直接执行，否则就先放入 change buffer 中，进行索引合并，将多个 DML 合并到一个操作中。



#### double write

double write 保证写入的安全性。InnoDB 缓冲池中刷出的脏页在写入数据文件前，会先将脏页写入 double write buffer，然后分两次写入到共享表空间中，再将数据页写到数据文件中去。即使页损坏了，也可以通过副本还原出之前的页，再通过 redo log 进行恢复。



#### adaptive hash index

InnoDB 可以监控索引的所有，如果发现查询可以通过建立索引得到优化，就会自动完成这件事，可以通过 innodb_adaptive_hash_index 参数来控制，默认为开启。