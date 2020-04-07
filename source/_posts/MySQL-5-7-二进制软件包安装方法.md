---
title: MySQL 5.7 二进制软件包安装方法
date: 2018-10-17 16:23:41
tags:
  - MySQL
categories:
  - 技术
---

MySQL 5.7 是目前最成熟的版本，性能较 5.6 有非常大的提升，又因为在官方软件包中所有的功能已经配置完毕，使用起来非常方便，所以本篇文章介绍 MySQL 5.7 版本的二进制软件包安装方法，步骤如下。





<!-- more -->

### 1. 下载并解压软件包

进入 [MySQL官网](https://www.mysql.com) ，选择 MySQL Community Server 进入软件包下载页面，下载文件如 mysql-5.7.23-linux-glibc2.12-x86_64.tar.gz，放入 `/usr/local/` 。

对下载的软件包进行 MD5 校验，`md5sum mysql-5.7.23-linux-glibc2.12-x86_64.tar.gz` ，确保下载过程没有问题。

使用 tar 命令解压软件包：

```shell
$ tar -zxvf mysql-5.7.23-linux-glibc2.12-x86_64.tar.gz
```

可以创建软链接方便使用：

```shell
$ ln -s mysql-5.7.23-linux-glibc2.12-x86_64 mysql
```



### 2. 创建 mysql 用户

```shell
$ groupadd mysql
$ useradd -g mysql mysql -s /sbin/nologin
```



### 3. 初始化数据库

```shell
$ /usr/local/mysql/bin/mysqld --user=mysql --initialize
```

使用以上命令初始化数据库，注意命令行打印出的包含临时密码的日志，如 `[Note] A temporary password is generated for root@localhost: M4s+iJ/lLUdS` 。

> MySQL 从 5.7.18 开始不在二进制包中提供 my-default.cnf 文件，不需要 /etc/my.cnf 配置文件也能正常运行。



### 4. 启动 MySQL

通过 mysqld_safe 命令启动 MySQL，可以通过 ps 命令查看是否启动成功。

```shell
$ /usr/local/mysql/bin/mysqld_safe &
$ ps -ef | grep mysql
```



### 5. 修改 root 密码，创建非 root 账号

使用 mysql 命令通过初始密码登录数据库，修改 root 账号密码，创建新的非 root 账号，并通过 flush privileges 更新权限。

```shell
$ /usr/local/mysql/bin/mysql -pM4s+iJ/lLUdS
```

```mysql
mysql> SET PASSWORD = 'xxx';
mysql> ALTER USER root@localhost PASSWORD EXPIRE NEVER;
mysql> CREATE USER jiangyue@'%' IDENTIFIED BY 'xxx';
mysql> GRANT ALL PRIVILEGES ON *.* TO jiangyue@'%' identified by 'xxx';
mysql> flush privileges;
```



### 6. 关闭 MySQL

正常关闭 MySQL 使用如下命令，非正常关闭需要 kill 掉 MySQL 进程。

```shell
$ /usr/local/mysql/bin/mysqladmin -uroot -pxxx shutdown
```



### * root 密码遗失解决方法

强制关闭数据库，通过加 **跳过权限表** 参数重启数据库并进入。

```shell
$ ps -ef | grep mysql
$ kill -9 22975 22917
$ /usr/local/mysql/bin/mysqld_safe --skip-grant-tables &
$ /usr/local/mysql/bin/mysql
```

选择 mysql 数据库，给 root 用户设置新的密码并刷新权限。

```mysql
mysql> use mysql;
mysql> UPDATE user SET authentiation_string = password('xxx') where user = 'root';
mysql> flush privileges;
```

修改密码后去掉 --skip-grant-tables 参数重启数据库。



### * 用户权限管理

通常超级权限的用户（root 和 all privileges 权限用户）只能归 DBA 管理，创建用户时最好保证专库专账号。创建用户、分配权限、刷新权限的语法如下：

```sql
mysql> CREATE USER username@host IDENTIFIED BY 'password';
mysql> GRANT CREATE, ALTER, SELECT, INSERT, UPDATE, DELETE ON DB.table TO username@host IDENTIFIED BY 'password';
mysql> flush privileges;
```

其中 host 可以指定具体 IP，也可以指定 IP 网段，如 `192.168.1.%` 。