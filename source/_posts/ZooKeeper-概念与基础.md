---
title: ZooKeeper 概念与基础
date: 2018-05-02 15:23:45
tags:
- ZooKeeper
categories:
- Tech
---

信息飞速膨胀，很多应用无法依赖单个服务器处理庞大的数据量。由于分布式系统和应用可以提供更强的计算能力，还能更好地容灾和扩展，所以逐渐受到青睐。

在开发分布式应用时，通常需要花费大量时间和精力来处理异构系统中的协作通信问题。





<!-- more -->

## 什么是 ZooKeeper

ZooKeeper 专注于任务协作，能为大型分布式系统提供可靠的协作处理能力，简化开发流程，让开发人员更专注于应用本身的逻辑。

ZooKeeper 具有 C-S 结构，在使用时，开发的应用可以看做为连接到 ZooKeeper 服务器的客户端，通过客户端 API 进行操作，保障强一致性、有序性和持久性，具有实现同步原语的能力，提供简单的并发处理机制。常见的用法如：选举主节点、崩溃检测、存储元数据等。

注意，ZooKeeper 不适用于海量信息存储，设计应用时最好将应用数据与协同数据分开，使用数据库、分布式文件系统等存储应用数据。



## ZooKeeper 基础

由若干条指令组成，用于完成特定功能的过程称为原语。

为了保证灵活性，ZooKeeper 不直接提供原语，而是暴露出类似文件系统的 API，让开发人员通过 API 实现自己的原语，通常使用菜谱（ recipes ）来表示原语的实现。

ZooKeeper 操作和维护的为一个个数据节点，称为 znode，采用类似文件系统的层级树状结构进行管理。如果 znode 节点包含数据则存储为字节数组（byte array）。



### API 概述


ZooKeeper 暴露如下 API：

* `create /path data` 创建节点并包含数据


* `delete /path` 删除节点
* `exists /path` 检查节点是否存在
* `setData /path data` 设置节点数据
* `getData /path` 获取节点数据
* `getChildren /path` 获取子节点列表

ZooKeeper 不允许局部读取或写入数据，读取时会将节点数据全部读取，写入时会整个替换。



### 节点类型

创建 znode 时需要指定节点类型，节点的类型会影响其行为方式。

znode 分为持久节点和临时节点。持久节点只有通过调用 delete API 才能删除；临时节点在客户端崩溃或关闭与 ZooKeeper 服务器连接时就会被删除，调用 delete API 也能删除临时节点。

znode 可设置为有序节点。有序节点在创建时会分配一个整数序号追加到路径之后，如创建 /tasks/task- 有序节点，追加整数后节点路径为 /tasks/task-1 。

综上，znode 共有 4 种类型，分别为：持久（无序）、临时（无序）、持久有序和临时有序。 



### 监听通知

ZooKeeper 通常以远程服务的方式被访问，在数据不发生变化时频繁地访问代价较大，ZooKeeper 的通知机制可以代替客户端的轮训。

客户端通过设置监视点（watcher）向 ZooKeeper 注册需要接收通知的 znode，在 znode 发生变化时 ZooKeeper 向客户端发送消息。

通知机制是单次触发的操作，如需不断监听 znode 的变化，需要在接收到 znode 变更时设置新的监视点。

监视点有多种类型，如监控 znode 数据变化、监控 znode 子节点变化、监控 znode 创建或删除。



### 版本匹配

每个 znode 都有一个随数据变化而自增的版本号，多个写入操作会有条件的执行。写入操作会将版本号作为参数，当与服务器上版本号一致时才会调用成功，这点在多个客户端对同一个 znode 进行操作时很重要。



### 仲裁模式

只有一台单独的 ZooKeeper 服务器时为独立模式，状态无法复制。具有一组 ZooKeeper 服务器时为仲裁模式，仲裁模式下各个服务器间进行状态的复制，同时服务于客户端的请求。

仲裁模式下，如果让客户端等待所有服务器完成数据保存再继续，延迟问题就会很大，ZooKeeper 通过法定人数确定工作时必须有效运行的服务器最小数量。如果有 5 台服务器，任意 3 台服务器保存了数据，客户端就可以继续工作。其他 2 台服务器最终也会捕获到数据进行保存。

服务器的个数应该为奇数，如有 5 台服务器，则最多允许 2 台崩溃。假如服务器个数为偶数，允许崩溃的服务器数量不会增加，法定人数却更大，需要更多的确认操作。



## 开始使用 ZooKeeper

开始使用 ZooKeeper 之前，需要 [下载 ZooKeeper 发行包](https://zookeeper.apache.org/) 。解压 tar 格式的压缩文件，bin 目录下有启动 ZooKeeper 的脚本，conf 目录下存放着 ZooKeeper 的配置文件，lib 目录下存放着一些运行所需的第三方文件。



### 启动服务器

在 ZooKeeper 目录下执行以下命令启动 ZooKeeper 服务器。

```shell
$ bin/zkServer.sh start
```



### 启动客户端

```shell
$ bin/zkCli.sh
```



### 列出根节点下的所有节点

```shell
[zk: localhost:2181(CONNECTED) 0] ls /
[zookeeper]
```



### 创建节点

```shell
[zk: localhost:2181(CONNECTED) 1] create /test ""
Created /test
[zk: localhost:2181(CONNECTED) 2] ls /
[zookeeper, test]
```



### 删除节点

```shell
[zk: localhost:2181(CONNECTED) 3] delete /test
[zk: localhost:2181(CONNECTED) 4] ls /
[zookeeper]
```



### 退出客户端

```shell
[zk: localhost:2181(CONNECTED) 5] quit
Quitting...
2018-05-02 15:58:34,004 [myid:] - INFO  [main:ZooKeeper@684] - Session: 0x1630a0cdc400001 closed
2018-05-02 15:58:34,006 [myid:] - INFO  [main-EventThread:ClientCnxn$EventThread@519] - EventThread shut down for session: 0x1630a0cdc400001
```



### 关闭服务器

```shell
$ bin/zkServer.sh stop
```

