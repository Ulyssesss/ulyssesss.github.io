---
title: MySQL 主从复制
date: 2019-01-03 21:16:39
tags:
  - MySQL
categories:
  - 技术
---

主从复制也称为主从同步，将一台主机的数据复制到另一台或多台主机上，通过中继日志 relay log 来实现复制功能，是构建数据库高可用集群架构的基础。

复制过程中一台服务器充当主库 master，其余一个或多个服务器充当从库 slave。





<!-- more -->

## 主从复制功能

主从复制有如下功能：

* 实时备灾，从库随时接管有故障的主库
* 从库分担主库的读压力，读写分离
* 利用从库做备份，减少对业务的影响
* 利用从库完成 MySQL 平滑的版本升级



## 主从复制原理

主库把 SQL 请求记录到 binlog 日志中，从库的 IO thread 向主库请求 binlog 日志，主库通过 IO dump thread 传送 binlog，从库将得到的 binlog 写入中继日志 relay log，然后 SQL thread 通过中继日志重做 SQL 语句。



## 复制中的重要参数

* log-bin 搭建主从复制必须开启 binlog
* server-id 同一组主从架构中的唯一标识
* server-uuid 由 MySQL 自动生成，每台机器不一样，在数据目录下的 auto.cnf 中
* read-only 设置从库为只读状态（对 super 账号没有效果）
* super-read-only 设置 super 账号是否开启只读状态
* binlog-format 二进制日志格式，必须使用 row 模式（默认 row）
* log_slave_updates 将源于主库的数据变更信息记录到从库 binlog 中
* todo



## 异步复制

异步复制为 MySQL 默认的复制方式，主库写入 binlog 后即可成功返回给客户端，无须等待 binlog 传递给从库。

> 异步复制的缺点是主库一旦发生宕机在高可用架构下做主备切换可能导致数据丢失。

以下为基于 binlog 和 position 的方式（非 GTID）搭建一主一从架构的步骤：

1. 修改主库和从库的 server-id，确保两者不一致。
2. 主库开启 binlog 功能（配置 log_bin）。
3. 从库最好开启 log_slave_updates 参数，让从库写 binlog，方便扩展。
4. 确保 binlog 格式为 row（默认为 row）。
5. 在主库中创建主从复制账号。

```mysql
mysql> create user 'bak'@'%' identified by 'bak123';
mysql> grant replication slave on *.* to 'bak'@'%';
mysql> flush privileges;
```

6. 通过 mysqldump 导出主库数据。

```shell
$ /usr/local/mysql/bin/mysqldump --single-transaction -uroot -pxxx --master-data=2 -A > all.sql
```

> --master-data=2 会在导出文件中记录当前使用的 binlog 文件及 position 号，并注释掉 change master 语句，change master 语句后续会手动执行。

7. 从库导入主库数据。

```shell
$ /usr/local/mysql/bin/mysql -uroot -pxxx < all.sql
```

8. 在从库执行配置主从的命令。

```mysql
mysql> change master to master_host='192.168.10.10',master_user='bak',master_password='bak123',master_port=3306,master_log_file='master-bin.000012',master_log_pos=1230;
```

9. 执行开始主从复制的命令并查看复制状态，从库 IO thread 和 SQL thread 均为 Yes 表示已经开始复制工作。

```mysql
mysql> start slave;
mysql> show slave status;
```

> 另 stop slave 用于关闭主从复制，reset slave all 用于清空从库复制配置。



## 半同步复制

MySQL 5.5 之后引入半同步复制功能，开启后主库需确保从库接收完 binlog 并写入到 relay log 中才会通知等待线程操作完毕。如等待超时则关闭半同步复制，自动转为异步复制，直到至少有一台从库通知主库已经接收到 binlog 为止。



### 半同步复制相关参数

* rpl_semi_sync_master_timeout

关闭半同步复制的超时时间，单位毫秒，默认 10000，可以调整得很大来禁止切换异步复制，以提高复制安全性。

* rpl_semi_sync_master_wait_point

用于控制半同步模式下事务的提交方式，包含 AFTER_COMMIT 和 AFTER_SYNC 两个值，AFTER_COMMIT 即将 binlog 传送给从库同时主库提交事务，AFTER_SYNC 即收到从库反馈后主库再提交事务。5.7 默认为 AFTER_SYNC，能够确保主库上提交的事务都已经同步给从库。

* rpl_semi_sync_master_wait_for_slave_count

控制主库需等待接收写入成功反馈的从库个数，当从库故障时可以通过该参数剔除故障从库。



### 半同步复制搭建过程

半同步复制的搭建基于异步复制，搭建完异步复制后只需安装半同步复制插件即可，以下为详细步骤：

1. 主库安装半同步复制插件并开启半同步复制功能。

```mysql
mysql> install plugin rpl_semi_sync_master soname 'semisync_master.so';
mysql> set global rpl_semi_sync_master_enabled=on;
mysql> show variables like '%semi%';
mysql> show plugins;
```

2. 从库安装半同步复制插件并开启半同步复制功能。

```mysql
mysql> install plugin rpl_semi_sync_slave soname 'semisync_slave.so';
mysql> set global rpl_semi_sync_slave_enabled=on;
mysql> show variables like '%semi%';
mysql> show plugins;
```

> 为了开机启动半同步复制功能，可以在主库和从库的配置文件 my.cnf 中分别配置 rpl_semi_sync_master_enabled=on 和 rpl_semi_sync_slave_enabled=on。

3. 重启从库 IO 线程并检查主库和从库的。

```mysql
## 从库重启 IO 线程
mysql> stop slave io_thread;
mysql> start slave io_thread;

## 检查主库半同步复制状态，注意 Rpl_semi_sync_master_status 和 Rpl_semi_sync_master_clients
mysql> show global status like '%semi%';

## 检查从库半同步复制状态，注意 Rpl_semi_sync_slave_status
mysql> show global status like '%semi%';
```

> 可以通过 `show global status like '%semi%'` 来查看主库接收从库事务回复的成功失败次数，分别为 Rpl_semi_sync_master_yes_tx 和 Rpl_semi_sync_master_no_tx。

至此 MySQL 半同步复制搭建成功。



### 半同步复制和异步复制切换

半同步复制状态下，如等待超时则关闭半同步复制，自动转为异步复制，当主库接收到来自从库的 binlog 反馈后会重新转为半同步复制。

比如在半同步复制状态下关闭从库，在主库中写入数据，写入数据的 SQL 会等待至半同步超时然后返回，此时通过 `show global status like '%semi%'` 可以看到主库已经关闭了半同步复制。

重新启动从库，如果从库配置了 `rpl_semi_sync_slave_enabled=on` ，则启动后主库从库自动转为半同步复制模式，否则会依然处于异步复制模式，需手动开启从库半同步复制然后重启从库 IO 线程才会转为半同步复制。



## 并行复制

MySQL 5.6 开始支持基于库级别的并行复制，及 slave_parallel_type=database。MySQL 5.7 中实现了基于组提交的并行复制，即从库可以通过多个 workers 线程并发执行 relay log 中的事务。

要开启基于组提交的并行复制，需要设置 slave_parallel_workers > 0，一般设置为 8 或 16，并将 slave_parallel_type 设置为 LOGICAL_CLOCK。

并行复制可以有效降低主从延迟，其他可以降低主从延迟的方法有采用 Percona 的 percona-xtradb-cluster (PXC) 架构多节点写入，选择合适的分库分表策略，避免无效 IO，使用高速磁盘，适当调整 buffer pool 大小等。