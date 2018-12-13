---
title: MySQL 表管理
tags:
  - MySQL
categories:
  - Tech
date: 2018-12-11 17:02:32
---

MySQL 其实就是一个包含很多表的集合，表中主要的数据类型可以分为整型、浮点型、字符类型和日期时间类型。MySQL 中数值类型可以指定是否区分正负和用零填补。选择数据类型时应符合最小、最合适的原则。





<!-- more -->

## 整型

MySQL 中整型类型有 tinyint、smallint、mediumint、int 和 bigint，分别占 1、2、3、4 和 8 个字节，其中 int 和 tinyint 最常用。

数据表中经常使用 id 字段作为主键，一般选择 int 类型，int unsigned 数值范围可达 42 亿，一般情况下足够业务上的应用。

需要注意的是，int(4) 和 int(11) 都占 4 个字节的空间，括号中的数字表示显示宽度，而非所占的字节数。如果类型定义为 int(n) zerofill，不足 n 位时会用 0 补充。



## 浮点型

浮点型包含 float、double 和 decimal，其中 float 占 4 个字节；double 占 8 个字节；decimal(m,d) 中 m 指最大位数（精度），范围是 1 到 65，d 是小数点右边的位数（刻度），范围为 0 到 30，且不得大于 m，d 省略时默认值为0，m 省略时默认值为10。



## 时间类型

时间类型包含 date、time、year、datetime 和 timestamp。

* date 占 3 个字节，范围为 1000-01-01 至 9999-12-31。
* time 占 3 个字节，范围为 -838:59:59 至 838:59:59。
* year 占 1 个字节，范围为 1901 至 2155。
* datetime 在 5.6 之前占 8 个字节，5.6 之后占 5 个字节，范围为 1000-01-01 00:00:00 至 9999-12-31 23:59:59。
* timestamp 占 4 个字节，范围为 1970-01-01 00:00:00 至 2038 年。

datatime 可用范围比 timestamp 大，仅比 timestamp 多占 1 个字节的空间，生产环境可以优先使用 datetime。或者也可以使用 int 来存储时间，通过 unix_timestamp 和 from_timestamp 函数转换使用。

datetime 和 timestamp 在 5.6 之后均支持自动更新为当前时间 ON UPDATE CURRENT_TIMESTAMP。



## 字符串类型

字符串类型包含 char、varchar、tinyblob、tingytext、blob、text、mediumblob、mediumtext、longblob 和 longtext。

* char 用于定长字符串，最多可容纳 255 个字符，没达到定义位数时尾部用空格不全存入表中，超过时截断。
* varchar 用于变长字符串，最多占 65535 个字节，没达到定义位数不会补空格，超过时截断。

> text、blob（二进制形式长文本）这种存储大量文字或图片的大数据类型最好不要与业务表放在一起。
>
> 不确定字段需要存储多少字符时使用  varchar 可以节约磁盘空间，提高存储效率。
>
> varchar(100) 中 100 表示最大字符数，UTF8 字符集下存储空间 为 100 * 3 + 2 个字节。字节数小于 255 时用 1 个字节记录长度，超过 255 时用 2 个字节记录长度。
>
> 存储 IPv4 时可以使用 int 类型，通过 inet_aton 和 inet_ntoa 函数来转换。



## 字符集

MySQL 中字符集包括 character 字符集 和 collation 校对规则，字符集用来定义数据字符串的存储方式，校对规则定义比较字符串的方式。

常见字符集包括 GBK、Latin1、UTF8、UTF8mb6：Latin1 目前已经不适用了；GBK 一个字符占 2 个字节，通用性没有 UTF8 好；UTF8 一个字符占 3 个字节；UTF8mb4 是 UTF8 的超集，一个字符占 4 个字节，可以存储 emoji 符号。建议使用 UTF8mb4。

> 临时修改数据库字符集，可以在数据库命令执行 set names，如 set names utf8。



## 碎片

在表中删除大量的数据后，数据文件的大小可能并没有减小。

delete 删除数据，MySQL 不会把数据文件真实删除，而是将数据文件的标示位删除，也不会整理数据文件，所以不会释放空间。新数据写入时，MySQL 会再次利用这些区域，但无法彻底占用。

删除操作会产生数据碎片，碎片会占用磁盘空间，扫描全表时也会扫描碎片部分，并且在读取效率方面比正常的空间低很多。

通过 `show table status like '%table_name%'` 语句可以查看表的状态，碎片大小 = 数据总大小 - 实际表空间文件大小，数据总大小 = data_length + index_length，实际表空间文件大小 = rows * avg_row_length。

innodb 引擎的表可以通过 `alter table table_name engine = innodb` 来重新整理全表数据，缺点是需要给整个表加写锁，并且需要一些时间。备份原表数据，删除整个表，重新导入到新表中也可以清理掉碎片。



## 库表常用命令

* `use database` 选择数据库
* `show databases` 查看全部数据库
* `show tables` 查看当前库所有的表
* `create database database_name` 创建数据库
* `drop database database_name` 删除数据库
* `create table table_name (字段列表)` 创建表
* `drop table table_name` 删除表
* `delete from table_name (where)` 、`truncate table table_name` 删除表内数据
* `insert into table_name (字段列表) values (对应字段的值)` 插入数据
* `update table_name set 字段名 = 值 (where)` 更新表内数据
* `select * from table_name (where)` 查看表内数据
* `show create table table_name` 查看建表语句
* `desc table_name` 、`describe table_name` 查看表结构
* `show table status` 获取表基础信息
* `show index from table_name` 查看当前表索引
* `show full processlist` 查看数据库当前连接

