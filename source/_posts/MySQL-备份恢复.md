---
title: MySQL 备份恢复
date: 2019-01-02 17:19:08
tags:
  - MySQL
categories:
  - 技术
---

MySQL 的备份方式按备份后的内容可分为全量备份和增量备份，按服务的运行状态可以分为冷备（停库）和热备（非停库），其中热备又可分为逻辑备份和裸文件备份。





<!-- more -->

## 冷备及恢复

冷备即数据库关闭状态下的备份，数据完整、过程简单且恢复速度快，但会影响业务进行，一般用于非核心业务。

备份时首先停止 MySQL 服务，然后复制整个数据目录到远程备份机或本地，恢复时用已备份的数据目录替换原有的目录并重启 MySQL 服务。相关命令如下：

```shell
## 关闭 MySQL
$ /usr/local/mysql/bin/mysqladmin -uroot -p{password} shutdown

## 上传数据目录至目标机器
$ scp -r {data directory} {user}@{ip}:{target directory}

## 启动 MySQL
$ /usr/local/mysql/bin/mysqld_safe &
```



## 热备及恢复

热备即数据库运行状态下的备份，不影响现有业务正常进行。

热备分为逻辑备份和裸文件备份，逻辑备份即备份 SQL 语句，恢复时执行备份 SQL，裸文件备份即在底层复制数据文件。



### 逻辑备份

逻辑备份可以通过 mysqldump、select ... into outfile 及 mydumper 三种方式实现。



#### mysqldump

mysqldump 为 MySQL 自带的命令工具，是最基础的备份工具。备份时先从 buffer 中找到需要备份的数据进行备份，如 buffer 中没有则将磁盘中的数据调入 buffer，然后进行备份，最后形成一个可编辑的备份文件。恢复数据时通过 mysql 命令工具进行。

可以通过 `/usr/local/mysql/bin/mysqldump --help` 查看使用说明，以下为一些核心参数：

* --single-transaction

用于保证 InnoDB 一致性，配合 RR 隔离级别使用，直到备份结束，不会读取到本事务开始之后提交的数据。

* --all-database（-A）

备份所有数据库。

* --master-data

在备份出的文件中添加 change master 语句（用于主从复制），该参数包含 1 和 2 两个值，1 时 change master 语句正常，2 时会将 change master 语句注释掉。

* --dump-slave

用于在搭建新从库，在库备份数据，1 时 change master 语句正常，2 时将 change master 语句注释掉。

* --no-create-info（-t）

只备份表数据，不备份表结构。

* --no-data（-d）

只备份表结构，不备份表数据。

* --complete-insert（-c）

使用完整的 insert 语句，包含表中的列信息，可以提高插入效率。

* --database（-B）

指定备份的数据库，如 `mysqldump -uroot -pxxx --database db1 db2` 。

* --where={condition}（-w）

按条件备份数据。

使用方法如下：

```shell
## 备份全库
$ /usr/local/mysql/bin/mysqldump -uroot -p{password} -A > all_20190102.sql

## 恢复全库
$ /usr/local/mysql/bin/mysql -uroot -p{password} < all_20190102.sql

## 备份db1数据库
$ /usr/local/mysql/bin/mysqldump -uroot -p{password} db1 > db1_20190102.sql

## 恢复db1数据库
$ /usr/local/mysql/bin.mysql -uroot -p{password} db1 < db1_20190102.sql

## 备份db1数据库中表t1
$ /usr/local/mysql/bin/mysqldump -uroot -p{password} db1 t1 > t1_20190102.sql

## 恢复db1数据库中表t1 (导入写库名即可)
$ /usr/local/mysql/bin.mysql -uroot -p{password} db1 < t1_20190102.sql

## 备份db1数据库表t1中id大于10的记录
$ /usr/local/mysql/bin/mysqldump -uroot -p{password} db1 t1 --where="id>10" > t.sql
```



#### select ... into outfile

select ... into outfile 的恢复速度比 insert 插入要快，但只能备份数据，不能备份表结构。

通过 select ... into outfile 备份时，先把备份的数据导出到一个文本文件中，然后通过 load data  方式恢复数据，使用方法如下：

```mysql
mysql> select * from {table name} into outfile '{target file}';
mysql> load data infile '{target file}' into table {table name};
```

> 如导出时报 The MySQL server is running with the --secure-file-priv option so it cannot execute this statement 错误，可以通过 `show variables like '%secure%'` 语句查看相关配置。
>
> `secure_file_priv = NULL` 表示不允许导出到文件，可以在配置文件 my.cnf 中 mysqld 模块下添加 `secure_file_priv =` ，表示允许导出到任意路径下的文件。



#### mydumper

mydumper 是一个高性能多线程备份工具，备份速度远高于单线程的 mysqldump，数据还原时使用 myloader 工具。

mydumper 需额外安装，本篇文章不做详细介绍。



### 裸文件备份

裸文件备份在底层复制数据文件，比逻辑备份速度更快。

Percona 公司的开源项目 XtraBackup 能够对 InnoDB 数据库进行裸文件热备，速度快且安全可靠，备份过程不会锁表，不影响业务。本篇文章不对 XtraBackup 做详细介绍。