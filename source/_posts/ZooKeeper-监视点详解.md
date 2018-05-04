---
title: ZooKeeper 监视点详解
date: 2018-05-04 14:52:03
tags:
- ZooKeeper
categories:
- Tech
---

应用程序需要知道 ZooKeeper 节点状态的情况很常见，如主从模式中从节点需要知道主节点是否崩溃。

轮训的方式很低效，尤其是在期望变化频率很低时。

ZooKeeper 提供了监视点（watch）这一处理节点变化的重要机制，客户端对指定 znode 节点注册通知请求，在节点状态发生变化时就会收到一个单次的通知。





<!-- more -->

## 单次通知

监视点是一个单次的触发器，应用对某个节点注册了一个监视点，会在该节点第一次触发事件时向应用发送通知。

监视点与会话关联，当客户端与一台 ZooKeeper 服务器断开连接后，会连接到集合中另一台服务器，同时将未触发的监视列表发送到服务器。如果所监视的节点已经发生变化，会向客户端发送通知，否则会在新的服务器上注册监视点。

单次触发的通知可能会丢失事件，比如在收到通知、注册新的监视点之间的这段时间所发生的事件。不过丢失事件通常不会造成影响，节点变化可以通过注册新监视点时读取节点状态来捕获。



## 设置监视点

ZooKeeper 中的所有读操作均可设置监视点，包括 getData、getChildren 和 exists。

使用监视点需要实现 Watcher 接口，重写 void process(WatchedEvent event) 方法。其中 WatchedEvent 包含 ZooKeeper 的会话状态（KeeperState）和事件类型（EventType）。

事件类型包括 NodeCreated、NodeDeleted、NodeDataChanged、NodeChildrenChanged 和 None。前 3 个类型涉及单个 znode 节点，第 4 个即 NodeChildrenChanged 涉及所监视节点的子节点。None 表示 ZooKeeper 会话状态发生变化。

| 事件类型            | 设置监视点方式                        |
| ------------------- | ------------------------------------- |
| NodeCreated         | 通过 exists 调用设置监视点            |
| NodeDeleted         | 通过 exists 或 getData 调用设置监视点 |
| NodeDataChanged     | 通过 exists 或 getData 调用设置监视点 |
| NodeChildrenChanged | 通过 getChildren 调用设置监视点       |



## 原子操作

ZooKeeper 在 3.4.0 版本添加了原子操作的特性，通过 multiop 可以原子性地执行多个 ZooKeeper 操作，这些操作要么全部成功，要么全部失败。

使用 multiop 时，首先创建出包含 ZooKeeper 操作的 Op 对象，再把多个 Op 对象放到 Iterable 类型的对象中供客户端调用，如：

```java
Op deleteZnode(String path) {
    return Op.delete(path, -1);
}

Op createZnode(String path, String data) {
    return Op.create(path, data.getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL);
}

List<OpResult> results = zk.multi(Arrays.asList(deleteZnode("/a"), createZnode("b", "data")));
```



multi 方法同样有异步版本，同步和异步方法的定义如下：

```java
public List<OpResult> multi(Iterable<Op> ops) throw InterruptedException,KeeperException;
public void multi(Iterable<Op> ops, MultiCallback callback, Object context);
```



Transaction 封装 multi 方法，提供了更简单的调用方式，可以通过创建 Transaction 对象来添加操作、提交事务。

```java
List<OpResult> results = zk.transaction().delete("/a", -1).create("/b", "data".getBytes()
        , ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT).commit();
```



multi 提供的另一个功能是检查 znode 的版本号，在输入的 znode 节点与版本号不匹配时，multi 会调用失败。

```java
public static Op check(String path, int version);
```



## 消除隐藏通道

ZooKeeper 的状态会在所有服务器间互相复制，服务端对状态变化的顺序会达成一致，并使用相同的顺序对状态进行更新。但事实上服务端很少同时执行更新操作，如果通过隐藏通道进行通信可能会出现状态不一致的现象。

比如 Client-1 连接到 Server-1，Client-2 连接到 Server-2，/z 节点中的数据由 a 变为 b。

Server-1 更新状态会向 Client-1 发送通知，Client-1 收到通知后向 Client-2 发送 /z 节点变化的消息，Client-2 再通过 Server-2 进行操作，如果此时 Server-2 还没有对 /z 节点的状态进行更新，就会读取到过期的数据。

Client-1 向 Clinet-2 发送消息，就是隐藏的通道，会导致错误发生，正确的做法是 Client-2 只通过 ZooKeeper 服务端接收消息，消除隐藏通道。



## 避免在同一个节点设置过多监视点

当节点发生变化时，会向设置监视点的客户端发送消息，如果对同一节点设置过多的监控点，就会在节点状态变更时出现发送消息的高峰，可能会对性能造成影响。

条件允许的话，最好对节点设置少量的监视点，理想情况下只设置一个。

比如多个客户端争相获取一个锁：可以一个客户端通过创建临时节点获取锁，其他客户端对这个节点设置监视点；也可以改为让客户端创建有序临时节点，最小序号的客户端获取锁，其他客户端只对比自己序号小 1 的节点设置监视点。

一个监视点会占用服务端约 300 字节的内存，开发时需要注意监视点数量。