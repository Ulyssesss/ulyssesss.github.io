---
title: MySQL 数据库文件
tags:
  - MySQL
categories:
  - Tech
date: 2018-12-03 21:29:36
---

本篇文章介绍 MySQL 中各种类型的文件，包含配置文件 my.conf、错误日志 error.log、慢查询日志 slow log、全量日志 general log、二进制日志 binlog、审计日志 audit log、套接字文件 socket、进程文件 pid ，及存储引擎层面的 redo log 和 undo log。





<!-- more -->

## 配置文件

通过命令 `mysql --help | grep my.cnf ` 可以快速查找 MySQL 配置文件的路径。

MySQL 会按照 `/etc/my.cnf` -> `/etc/mysql/my.cnf` ->  `/usr/local/mysql/etc/my.cnf` ->  `~/.my.cnf` 的顺序依次查找配置，如果多个配置文件中包含对某项的配置，以读取到最后一个配置文件中的参数值为准。如果想指定默认的配置文件，可以在启动时通过 --defaults-file 参数指定。

配置文件 my.cnf 中，针对客户端的配置写在 client section，针对服务端的配置写在 mysql section，如

```properties
[client]
port = 3306

[mysqld]
wait_timeout = 300
```

以下介绍一些核心参数，可以通过 `show variables like '%param_name%'` 来查看配置的参数。

* innodb_buffer_pool_size

缓冲池大小，用于缓存被访问过的表和索引，提升处理速度，默认为 128M。如果服务器只跑数据库一个应用，可以设置为物理内存的 50 ~ 80%，从 MySQL 5.7 开始支持在线修改。



* innodb_buffer_pool_instance

缓冲区划分的区域个数，默认为 1，MySQL 5.6.6 之后可以调整为多个，以提高并发性，仅在 innodb_buffer_pool_size 大于 1G 时生效。



* innodb_buffer_pool_load_at_startup 和 innodb_buffer_pool_dump_at_shutdown

启动时加载缓冲池、关闭时 dump 缓冲池，默认为关闭，两个参数都开启可以在宕机后快速将热数据加载回来。



* innodb_data_file_path

指定系统表空间文件的路径和 ibdata1 文件的大小，默认10M，遇到高并发事务时会有影响，可以调整为 1G。



* innodb_flush_log_at_trx_commit、sync_binlog 和 innodb_max_dirty_pages_pct

innodb_flush_log_at_trx_commit 参数控制 redo log 写入策略：0 表示每个 1s 写入文件并刷盘，但每次事务提交不会触发写入；1 表示每次事务提交都写入并刷盘；2表示每次事务提交写入文件但不刷盘。

sync_binlog 参数控制 binlog 写入磁盘的策略：0 表示不出发任何同步磁盘指令，让文件系统自行决定，N 表示每 N 次事务发出一次同步指令。

innodb_max_dirty_pages_pct 控制脏页在 buffer pool 中的最大占比，到达上限后出发脏页刷新，默认为 75%，为了上产环境的 TPS 性能可以改为 25 ~ 50%。



* innodb_thread_concurrency

InnoDB 内核最大并发线程数，默认为 0，表示不受限制。如果数据库压力过大，可以调整为服务器逻辑 CPU 核数的两倍，之后再慢慢增大。



* interactive_timeout、wait_timeout

服务器关闭交互式和非交互式连接之前的最长等待时间，默认为 28800s（8h），建议调整为 300 ~ 600s。



* innodb_flush_method

InnoDB 数据、redo log 的打开刷写模式，类 Unix 系统下默认为 fsync，如果使用高速 IO 设备，O_DIRECT 是更好的选择。



* innodb_old_blocks_time、innodb_old_blocks_pct

InnoDB 缓冲池内部由 LRU 链表管理，分为 young pages list 和 old pages list，当未被访问时间超过 innodb_old_blocks_time 时被移入 old pages list，默认为 1000ms。

innodb_old_blocks_pct 指定 old pages list 占总体的百分比，默认为 37%，可以适当减小来保证更多的热数据不被冲掉。



* transaction_isolation

事务隔离级别，可选值包括 READ-UNCOMMITTED、READ-COMMITTED、REPEATABLE-READ 和 SERIALIZABLE，默认为 REPEATABLE-READ（可重复读）。



* innodb_open_files

InnoDB 可以同时打开的 .bd 文件个数，最小值为 10，默认 300，应该适量增大。



* innodb_log_buffer_size、innodb_log_file_size

innodb_log_buffer_size 为日志缓冲大小，可以通过 `show global status like '%Innodb_log_waits%'` 查看等待日志刷新的次数，如果大于 0 且持续增长，就可以适当增大，默认为 16777216（16M），取值范围为 16 ~ 64M。

innodb_log_file_size 为 redo log 大小，过大会导致实例恢复时间长，过小会导致 redo log 频繁切换。



* innodb_log_files_in_group

redo log 文件组中日志文件的数量，默认为 2。



* max_connections

MySQL 最大连接数，默认为 151，对于并发连接很多的应用远远不够，应适当增大，同时注意数据库承受大量连接的压力。



* expire_logs_days

binlog 过期时间，单位为天。



* slow_query_log

慢查询日志开关，生产环境应该开启慢查询日志。



* long_query_time

慢查询时间，如果 SQL 语句的执行时间超过该值就会记录到慢查询日志中，单位为秒。



* log_queries_not_using_indexex

如果 SQL 没有用到索引，根据该配置决定是否记录到慢查询日志中。



* server-id

搭建主从环境时同一主从结构中的唯一标识。



* binlog_format

binlog 日志格式，有 statement、row 和 mixed 三种，生产环境使用 row 更安全，不会出现跨库复制丢数据的情况。



* lower_case_table_names

表名是否区分大小写，默认为区分。



* innodb_fast_shutdown

定义 InnoDB 引擎的表关闭时的行为，值可选 0、1 和 2，默认为 0。其中 0 表示需要执行 purge all、merge change buffer、flush dirty pages 操作，关闭最慢但重启最快；1 表示仅执行 flush dirty pages；2 表示上述操作都不会执行，仅将日志写入文件，重启时会执行 recovery 操作。



* innodb_status_output、innodb_status_output_locks

将数据库监控信息写到 error log 的开关，默认关闭，如开启会导致错误日志增长过快。



* innodb_io_capacity

影响刷新脏页和插入缓冲的数量，默认为 200，使用高速磁盘可以适当提高。



* auto_increment_increment、auto_increment_offset

指定自增字段每次递增的量和开始的值。



## 参数类型

MySQL 中的参数分为动态参数和静态参数。动态参数可以通过 set global 或 set session 命令在线修改，效果分别为全局生效和只针对当前会话生效；静态参数在修改时会报 read only variable 错误，只能修改配置文件并重启数据库。



## 错误日志

错误日志一般在数据目录下，记录 MySQL 在启动、运行、关闭过程中出现的问题，另外还有一些警告信息。



## binlog

binlog 和 redo log 同样记录对数据执行更改的操作，不会记录 select 和 show 这样的语句。

binlog 可以完成主从复制，主库将所有修改数据的操作记录到 binlog 中，通过网络发送给从库服务器，从而完成同步。

binlog 也可以进行数据恢复，使用 mysqlbinlog 命令可以实现基于时间点和位置的恢复操作。

在配置文件中使用 log_bin = [filename] 开启 binlog，默认存储在数据目录下，后缀为序列号。

> 配置文件中开启 binlog 必须同时指定 server-id，否则会启动失败。

### 相关参数

* max_binlog_size 单个 binlog 最大值，超过该值或重启会生成新的 binlog，默认为 1 GB。

> 生产环境一般控制 binlog 生成的最小间隔时间在 2 ~ 5 分钟，可以调整为 256 MB。

* binlog_cache_size 未提交事务缓存大小，基于会话，默认为 32 KB。

> 设置过小会使用磁盘上的临时文件，可以通过 show global status 查看 binlog_cache_use 和 binlog_cache_disk_use 的使用情况判断参数是否合适。生产环境一般设置为 1 ~ 4 MB。

* binlog_format 日志格式，可选值有 statement、row 和 mixed 三种，默认为 row。row 的优点是安全可靠，缺点是产生的日志量大。

> 使用 mysqlbinlog 工具可以导出转换格式后的二进制日志，`mysqlbinlog --no-defaults -v -v --base64-output=decode-rows binlog-file > target-file` ，其中 --no-default 表示不从任何配置文件中读取配置，-v 表示显示具体的执行信息，两个 -v 可以显示列的数据类型及注释，--base64-output 指定转换格式。

* sync_binlog，影响 binlog 磁盘刷新，前面提到过。
* expire_logs_days，binlog 过期时间，单位天，可以设置长一点的时间。
* binlog-do-db、binlog-ignore-db，指定需要写入和忽略日志的库，默认为空，写入所有库的日志。
* log_slave_updates，在搭建 m -> s1 -> s2 这样的主从架构时，需要在 s1 配置为 1，才能实现 s1 到 s2 的同步。
* binlog_checksum，指定 binlog 的校验算法，可选 none 和 crc32，默认 crc32。
* binlog_row_image，binlog 记录策略，可选 full、minimal 和 noblob，分别表示全记录、只记录要修改的列和记录除 blob 和 text 以外的列，默认为 full。



## slow log

慢查询日志会把超过 long_query_time 参数配置时间的所有 SQL 语句记录进来，long_query_time 默认 10 s。

可以使用 percona-toolkit 工具来分析慢查询日志。



## general log

全量日志，记录全部 SQL 语句，包含 select 和 show，一般不会开启，默认关闭。个别情况可以临时开启，用于故障检测。

log_output 参数指定全量日志的存储方式，可取 FILE、TABLE 和 NONE。FILE 可以方便地按条件检索，指定为 NONE 即使全量日志开启也不会记录日志，TABLE 会在 MySQL 创建一个 general_log 表。建议使用 FILE。



## audit log

审计日志，能够实时记录网络上的数据库活动，进行细粒度审计的合规性管理，通过对数据库行为的记录、分析和汇报来生成报告、事故追根溯源。

购买 MySQL 企业版才可以使用审计功能，也可以使用第三方开源插件 libaudit_plugin.so 完成审计工具。



## socket

MySQL 有网络连接和本地连接两种连接方式，mysql.sock 文件是服务器和本地客户端通信的 UNIX 套接字文件，默认位置为 /tmp/mysql.sock。



## pid

进程文件，记录应用的进程号，默认在 MySQL 数据目录下。



## redo log

记录事务操作变化，即数据被修改之后的值，用于异常宕机后的数据恢复。



## undo log

对记录的更新操作不仅会产生 redo 记录，也会产生 undo 记录，undo log 记录变更前的旧数据。默认记录在系统表空间 ibdata1 中，从 MySQL 5.6 开始可以使用独立 undo 表空间，避免将 ibdata1 文件弄大，也给部署不同 IO 类型的文件位置带来便利。



### 相关参数

* innodb_undo_directory，undo log 存储目录
* innodb_undo_logs 回滚段数量，默认为 128，每个回滚段最多存放 1024 个事务。
* innodb_undo_tablespaces 表示 undo tablespace 个数，默认为 0，最少为 2，保证在线 truncate 时至少有一个可用空间。
* innodb_max_undo_log_size，最大 undo tablespace 文件大小，默认 1 GB，超过时触发 truncate，truncate 后恢复为 10 MB，通过 innodb_undo_log_truncate 配置来开启或关闭该特性，默认关闭，并且仅支持独立 undo 表空间。
* innodb_purge_rseg_truncate_frequency，控制回收 undo log 频率，默认为 128，即 purge undo 轮询 128 次后进行一次 undo 的 truncate 操作。

