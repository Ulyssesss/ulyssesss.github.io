---
title: Redis 安装和基础数据结构
date: 2019-08-16 11:12:00
tags:
  - Redis
categories:
  - Tech
---

Redis 包含 5 中基础数据结构，分别为字符串 string 、列表 list 、字典 hash 、集合 set 和有序集合 zset 。

本篇文章简要介绍 Redis 的安装和这 5 种基础数据结构的使用。



<!-- more -->



### Redis 安装

官网下载源码压缩包，如 `redis-5.0.5.tar.gz` ，解压后编译即可完成安装。

```shell
$ cd redis-5.0.5
$ make

# 运行 Redis 服务器
# --daemonize yes 或在 redis.conf 中修改 daemonize 为 yes 表示后台运行
$ cd src
$ ./redis-server

# 运行命令行客户端
$ ./redis-cli
```





### 基础数据结构

Redis 中所有的数据结构都以唯一的 key 作为名称，通过 key 来获取相应的 value。不同数据结构类型的差异在于 value 的结构不一样。

list 、 set 、hash 和 zset 这四种数据结构为容器型结构，对其进行操作时，如果 key 不存在则先创建再操作，最后一个元素被删除时数据结构被删除。

Redis 中可以对 key 设置过期时间，到时间会自动删除 key ，常用于控制缓存的失效时间。

需要注意的是，只能对整个 key 设置过期时间，不能对某个子 key 设置过期时间，另外对已经设置了过期时间的 key 进行修改，过期时间会消失。



#### string

字符串是 Redis 中最简单的数据结构，可以动态地修改字符串，常见的用法如缓存用户信息。

string 的内部表示是一个字符数组，类似于 Java 中的 ArrayList ，通过预分配冗余空间来减少内存的频繁分配。

当字符串长度小于 1 MB 时，按现有空间加倍扩容，超过 1 MB 时每次扩容 1 MB ，最大长度为 512 MB 。

string 的操作命令如下：

```shell
# 设置 name 的值为 Jack
> set name Jack
OK

# 获取 name 的值
> get name
"Jack"

# 判断是否存在键 name
> exists name
(integer) 1

# 删除键 name
> del name
(integer) 1

# 批量设置
> mset name1 Jack name2 Tony name3 Jay
OK

# 批量获取
> mget name1 name2 name3
1) "Jack"
2) "Tony"
3) "Jay"

# 设置 5 秒后过期
> expire name1 5
(integer) 1

# 设置 key 且 5 秒后过期，相当于 set + expire
> setex name 5 Jack
OK

# 如果 key 不存在就创建
> setnx name Jack
OK

# 如果 value 为整数，可以对其进行自增操作
> set age 18
OK
> incr age
(integer) 19

# age 的值加 5（可传入负数表示减操作）
> incrby age 5
(integer) 24
```



#### list

Redis 中的列表相当于 Java 中的 LinkedList，是链表而非数组。插入和删除操作非常快，时间复杂度 O(1) ，但索引定位很慢，时间复杂度 O(n) 。

Redis 底层存储的不是简单的 LinkedList ，而是称为 QuickList 的结构。元素较少时使用连续的内存存储（ziplist 压缩列表），元素多时将多个 ziplist 组成链表，既满足快速插入删除的性能，又节约了空间。

Redis 列表常用做异步队列，将任务塞进列表，另一个线程从列表中轮询数据进行处理。

list 的操作命令如下：

```shell
# 向右插入一个或多个元素
> rpush books b1 b2 b3
(integer) 3

# 向左插入一个或多个元素
> lpush books b4 b5 b6
(integer) 6

# 获取列表长度
> llen books
(integer) 6

# 从左侧弹出一个元素
> lpop books
"b6"

# 从右侧弹出一个元素
> rpop books
"b3"

# 获取指定索引的元素
> lindex books 3
"b2"

# 截取指定索引内的列表，仅保留截取范围内的区间
# index 可以为负数，-1 表示倒数第一个元素，-2 表示倒数第二个元素
> ltrim books 1 2
OK

# 获取指定索引区间的元素
> lrange books 0 -1
1) "b4"
2) "b1"
```



#### hash

Redis 中的字典相当于 Java 中的 HashMap ，实现结构也与 Java 一致，即数组和链表的二维结构，一维 hash 数组位置碰撞时，将碰撞的元素用链表串起来。

Redis 中字典的值只能是字符串，另外 rehash 时采用了渐进式 rehash 策略。

渐进式 rehash 策略在执行时，会同时保留两个 hash 结构，查询时同时查询两个结构，由定时任务循序渐进地将旧 hash 中的数据转移到新 hash 中，全部转移完成用新的 hash 取代旧的，旧的 hash 被删除。

字典可用于按字段存储用户信息，从而不必获取全部的信息，不过 hash 结构的存储消耗要高于字符串。

hash 的操作命令如下：

```shell
# 对指定键插入 key - value ，更新操作返回 0
> hset books redis RedisBook
(integer) 1
> hset books java JavaBook
(integer) 1
> hset books java NewJavaBook
(integer) 0

# 获取指定字典的全部内容
> hgetall books
1) "redis"
2) "RedisBook"
3) "java"
4) "NewJavaBook"

# 获取字典长度
> hlen books
(integer) 2

# 获取字典中指定 key 的 value
> hget books redis
"RedisBook"

# 批量插入 key - value
> hmset books python PythonBook golang GolangBook
OK

# 对字典中单个 key 进行计数操作
> hset books number 1
(integer) 1
> hincrby books number 5
(integer) 6
```



#### set

Redis 中的集合相当于 Java 中的 HashSet ，键值对无序且唯一，内部实现相当于一个特殊的字典，value 都是 NULL 。

集合可用于存储活动中奖用户 ID ，保证一个用户不会中奖两次。

set 的操作命令如下：

```shell
# 向集合中插入一个或多个元素
> sadd books python
(integer) 1
> sadd books python
(integer) 0
> sadd books java golang
(integer) 2

# 获取集合中全部元素
> smembers books
1) "python"
2) "golang"
3) "java"

# 获取集合大小
> scard books
(integer) 3

# 判断元素是否在集合中
> sismember books java
(integer) 1

# 弹出一个元素
> spop books
"golang"
```



#### zset

Redis 中的有序集合 zset 既可以保证内部元素的唯一性，又可以给每一个元素赋予一个排序权重 score 。

有序集合可以用于存储学生成绩，元素即学生 ID ，score 为其考试成绩，可以按成绩排序获取名次。

zset 的操作命令如下：

```shell
# 向有序集合中插入元素
> zadd books 1 book1
(integer) 1
> zadd books 3 book2
(integer) 1
> zadd books 2 book3
(integer) 1

# 按 score 排序列出指定名次范围的元素
> zrange books 0 1
1) "book1"
2) "book3"

# 按 score 逆序列出指定名次范围的元素
> zrevrange books 0 1
1) "book2"
2) "book3"

# 获取集合大小
> zcard books
(integer) 3

# 获取指定 value 的 score
> zscore books book1
"1"

# 获取指定 value 的排名
> zrank books book1
(integer) 0

# 按 score 区间遍历集合
> zrangebyscore books 0 2
1) "book1"
2) "book3"

# 删除指定 value
> zrem books book3
(integer) 1
```