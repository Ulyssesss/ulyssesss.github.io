---
title: Redis 消息队列
date: 2019-08-20 18:14:08
tags:
  - Redis
categories:
  - Tech
---

Redis 可用作轻量级的消息队列，使用简单。

不过 Redis 不是专业的消息队列，缺少很多的高级功能，比如没有 ack 保证（ Acknowledgement ，消息确认机制）。如果对消息可靠性有较高的要求，就不适合使用 Redis 消息队列。



<!-- more -->



### 异步消息队列

Redis 中的 list 数据结构常用作异步消息队列，通过 lpush 或 rpush 指令操作入队列，通过 lpop 或 rpop 指令操作出队列，支持多个生产者和一组消费者（组内每个消费者拿到不同的列表元素），示例如下：

```shell
# 消息入队列
> rpush notify-queue java python golang
(integer) 3
> rpush notify-queue php
(integer) 4

# 消息出队列
> lpop notify-queue
"java"
> lpop notify-queue
"python"
```



#### 空队列处理

如果队列中的消息全部处理完成，pop 指令获取不到元素，客户端会陷入循环，浪费资源。

一种解决方法是让客户端线程 sleep ，每隔一段时间尝试一次获取消息，缺点是消息的消费会存在延迟。

更好的解决方法是使用 blpop 或 brpop 指令来阻塞读取消息，b 表示 blocking ，没有新消息时阻塞，来新消息时能够立即获取到消息。

需要注意的是，如果阻塞时间过长，Redis 服务端会主动断开连接以回收资源，blpop 或 brpop 会抛出异常，客户端消费者需要捕获异常并重试，示例如下：

```shell
# 读取消息，最长阻塞时间 5 s
> blpop notify-queue 5
1) "notify-queue"
2) "golang"
> blpop notify-queue 5
1) "notify-queue"
2) "php"

# 5 s 内未读取到消息，断开
> blpop notify-queue 5
(nil)
(5.04s)

# 生产消息
> rpush notify-queue C#
(integer) 1

# 阻塞 3.05 s 读取到消息
> blpop notify-queue 5
1) "notify-queue"
2) "C#"
(3.05s)
```



### 延时队列

延时队列可以通过 Redis 中的 zset 数据结构来实现，其中消息的内容作为 zset 的 value ，消息的到期处理时间作为 zset 的 score 。

使用时通过 zadd 指令生产消息，多个客户端线程通过 zrangebyscore 指令读取队列中的消息，再通过 zrem 指令来确定消息的所属消费者，成功删除消息的消费者执行消费逻辑，示例如下：

```shell
# 生产消息 zadd <key> <score> <value>
> zadd delay-queue 1566298823195 msg1
(integer) 1

# 获取消息 zrangebyscore <key> <min-score> <max-score> <offset> <count>
> zrangebyscore delay-queue 0 1566298924936 0 1
1) "msg1"

# 删除消息成功，执行消费逻辑 zrem <key> <value>
> zrem delay-queue msg1
(integer) 1

# 删除消息失败，消息已被其他消费者消费
> zrem delay-queue msg1
(integer) 0
```

> 示例中同一个任务被多个线程取到后，再通过删除决定从属关系，可以通过 lua 脚本将 zrangebyscore 和 zrem 一同在服务端进行原子化操作，从而节约资源。