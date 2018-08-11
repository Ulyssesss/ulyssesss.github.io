---
title: Java 并发概述
date: 2017-07-02 23:43:33
tags:
- Java
categories:
- Tech
---

操作系统拥有在同一刻运行多个程序的能力，并发执行的进程数不受 CPU 数目制约，系统将 CPU 的时间片分配给每个进程，给人并行处理的感觉。

而多线程扩展了多任务的概念，即一个程序同时执行多个任务（线程）。每个进程拥有自己的一套变量，而线程则共享数据。本篇文章将从线程、同步、执行器等多方面介绍 Java 并发。





<!-- more -->





## 线程

通常人们会提防长时间的计算，如在 GUI 或 web 项目中，如果需要执行一个比较耗时的任务，应该使用线程并发的方式快速响应。



### 线程的简单使用

创建单独的线程并执行任务很简单，需要将任务实现在 Runable 接口的 run 方法中，通过 Runable 创建 Thread 对象，然后通过 Thread 对象的 start 方法启动线程。



### 中断线程

当线程的 run 方法执行完毕或出现了没有被捕获的异常时，线程会中止。没有可以强制终止线程方法，但 interrupt 方法可以用于请求终止线程。

>  早起 Java 版本中，其他线程可以通过调用 stop 方法终止线程，不过已经被弃用，因为无法确定什么时候调用 stop 方法是安全的，suspend 方法同样被弃用，经常导致死锁。

每个线程都有一个 boolean 中断状态标志，调用 interrupt 方法会使中断状态被置位，可以不时地通过 Thread.currentThread 方法获取当前线程，然后通过 isInterrupted 方法判断线程的中断状态。

在一个被阻塞（sleep 或 wait）的线程上调用 interrupt 方法时会产生 InterruptedException，如果要对一个存在阻塞情况的线程发送终止请求，必须要处理 InterruptedException。



### 线程状态

线程存在以下 6 种状态：New、Runnable、Blocked、Waiting、Timed waiting 和 Terminated，调用 getState 可以获取当前状态。

使用 new 创建一个新线程时，线程还没有开始运行，状态是 New，一旦调用 start 方法，线程处于 Runnable 状态。

如果线程视图获取一个被其他线程持有的锁，线程会进入 Blocked 状态，直至其他线程释放锁且调度器允许该线程持有锁；而当线程等待另一个线程通知调度器一个条件时，会进入 Waiting 状态。有些方法包含超时参数，可以通过调用这些方法进入 Timed waiting 状态。

当线程执行完毕或出现没有被捕获的异常时线程被终止。



### 线程属性

Java 中每个线程都有优先级，默认情况线程继承父进程的优先级，可以通过 setProiority 方法提高或降低优先级。Java 线程的优先级取决于宿主机平台的线程实现机制，所以不要使程序的正确性依赖于优先级。

通过调用 setDaemon 方法可以将线程转换为守护线程，守护线程的唯一用途是为其他线程提供服务，当只剩下守护线程时虚拟机就退出了。守护线程永远不应该访问文件、数据库等固有资源，它可能会在任何时候发生中断。

线程在因为异常而死亡之前，会将异常传递到一个处理器，该处理器实现 Thread.UncaughtExceptionHandler 接口。可以通过 setUncaughtExceptionHandler 方法为线程安装处理器，也可以使用静态方法 setDefaultUncaughtExceptionHandler  为所有线程安装处理器。线程的处理器默认为空。





## 同步

在大部分多线程应用中，多个线程会共享数据，如果多个线程都对同一个对象进行修改，将会根据写入的顺序产生不同的结果。为了避免多线程引起的共享数据错误，需要使用同步存取，Java 提供了 synchronized 关键字和 ReentrantLock 类两种机制来确保任何时刻只有一个线程进入临界区。



### ReentrantLock

使用 ReentrantLock 时，需要在代码块前通过 lock 方法对代码块加锁，并在 finally 子句（确保抛出异常时能够释放）中通过 unlock 方法释放锁。一旦一个线程封锁了锁对象。其他线程无法通过 lock 语句而被阻塞，直到锁被释放。

ReentrantLock 是可重入的，即线程可以重复获得已经持有的锁，ReentrantLock 用一个计数器来跟踪 lock 的嵌套调用。

ReentrantLock 有公平策略，即锁倾向于等待时间最长的线程。公平策略会影响性能，默认不是公平的。



### Condition

有时进入临界区的线程需要在某些条件满足时才能执行，但这一线程刚刚获得了锁，其他线程无法在此时持有锁进而无法工作，所以需要条件对象 condition。

一个锁对象可以有多个条件对象，通过 newCondition 方法可以获得条件对象。当条件不满足时通过 await 方法阻塞线程并放弃锁，进入该条件的等待集。等待集中的线程在锁可用时不会解除阻塞，需要另一线程调用同一条件的 signalAll 方法才会变为可运行状态并开始竞争锁，重新获取锁后从 await 返回并从被阻塞的位置继续执行。

注意 signalAll 方法无法确保条件已经被满足，需要再次去检测条件。另外 signal 方法可以随机解除等待集中的一个线程，比 signalAll 更高效，但也存在危险（随机选择的线程有可能仍然不能运行）。



### synchronized

锁和条件对象提供了高度的锁定控制，然而在多数情况下不需要那样的控制。Java 中每个对象都有一个内部锁，如果一个方法用 synchronized 关键字声明，则锁对象将保护整个方法，即调用方法必须获得内部锁。

内部对象锁只有一个条件，通过 wait 方法可以将线程添加到等待集，notifyAll 或 notify 方法解除等待线程的阻塞状态。

synchronized 可用于静态方法，方法将获得类对象的内部锁。

通过对一个对象使用 synchronized 也可以获得锁，如：

```java
private Object lock = new Object();
public void test() {
    synchronized (lock) {
        ...
    }
}
```



### volatile

Java 提供了一种稍弱的同步机制，即 volatile 变量。变量声明为 volatile 之后，对该变量的操作不会与其他内存操作一起重排序，也不会被缓存在寄存器等地方，读取 volatile 变量时总会返回最新写入的值。

访问 volatile 变量不会加锁，线程不会阻塞，比 synchronized 更轻量级。



### 原子性

java.util.concurrent.atomic 包中有很多类使用了高效的机器级指令保证操作的原子性，如 AtomicInteger 提供了 incrementAndGet 和 decrementAndGet 方法以原子方式将整数自增、自减。

如果大量线程访问相同的原子值，性能会大幅下降，Java8 提供了 LongAdder 和 LongAccumulator 来解决这个问题。

LongAdder 包含多个变量，其总和为当前值。不同线程更新不同的加数，如果全部工作完成后才需要总和的值这种方法非常高效。

LongAccumulator 在构造器中可以提供这个操作及它的零元素，通过 accumulate 方法加入新的值，通过 get 方法获取当前值，最终得到 a1 op a2 op a3 ... 。



### 线程局部变量

有些类不是线程安全的，如 SimpleDateFormat，使用同步开销太大，在需要时创建新的 SimpleDateFormat 对象也有些浪费，可以为每个线程构造一个实例，如：

```java
public static final ThreadLocal<SimpleDateFormat> format =
            ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
String dateString = format.get().format(new Date()) ;
```



### tryLock

线程在调用 lock 方法获取锁时可能会发生阻塞，如果希望获取锁失败后立即返回，可以使用 tryLock 方法，成功获取锁后返回 true，否则返回 false。tryLock 可以使用超时参数，使阻塞时间不超过给定的值。



### 读/写锁

ReentrantLock 可以通过 readLock 和 writeLock 方法抽取出读锁和写锁，在多读取少写入的情况下可以仅使用写锁，允许读者线程共享访问。





## 阻塞队列

很多线程问题可以通过使用阻塞队列来方便、安全地解决，生产者线程向队列插入元素，消费者线程读取它们。当试图向已满队列添加元素或从空队列读取元素时，阻塞队列导致线程阻塞。

阻塞队列提供了 3 组对列满或空有不同响应方式的方法，其中 put、take 会阻塞，add、remove、element 会抛出异常，offer、poll、peek 返回 false 或 null。



## 线程安全的集合

多线程并发地修改一个数据结构，可能会破坏数据，如 HashMap。可以通过锁来加以保护，更好的方式是使用线程安全的集合。

java.util.concurrent 包提供了映射、有序集和队列的高效实现：ConcurrentHashMap、ConcurrentSkipListMap、ConcurrentSkipListSet 和 ConcurrentLinkedQueue。集合返回弱一致性的迭代器，即不一定能反映出所有的修改，但能保证同一个值不会返回两次。





## Callable 和 Future

Callable 和 Runnable 类似，只是任务执行的方法有一个参数化类型的返回值。

Future 保存异步计算的结果，将 Future 对象交给某个线程，在结果计算成功后可以通过 Future 对象获取结果。Future 对象的 get 方法会阻塞，直到计算完成。get 方法可以设置超时时间，超时会抛出 TimeoutException。

FutureTask 可以将 Callable 转为 Future 和 Runnable，如：

```java
Callable<Integer> callable = ...;
FutureTask<Integer> task = new FutureTask<>(callable);
new Thread(task).start();
Integer result = task.get();
```





## 执行器

如果程序中需要使用大量的生命期很短的线程，就应该使用线程池。线程池中包含空闲线程，可用于执行任务，任务完成后线程不会死亡，而是准备完成下一个任务。线程池还可以配置固定的线程数来限制并发线程总数。

Java 中 Executors 类提供了构建线程池的方法，其中 newCachedThreadPool 会在必要时创建新线程，空闲线程保留 60 秒；newFixedThreadPool 创建固定线程数的线程池，空线程会一直保留；newSingleThreadExecutor 创建只有一个线程的线程池，顺序执行提交的任务；newScheduledThreadPool 和 newSingleThreadScheduledExecutor 用于创建预定执行的线程池。

通过 submit 方法可以将 Runnable 或 Callable 对象提交给 ExecutorService，返回 Future 对象（对提交 Runnable 返回的 Future 调用 get 方法返回 null）。提交完全部任务应该调用 shutdown 方法，该方法会使执行器不再接收新任务，并在完成全部任务后终止线程。调用 showdownNow 方法会取消尚未开始的任务并试图中止正在运行的线程。

awaitTermination 方法需要传入一个超时时间，该方法会阻塞至全部任务执行完毕或到达超时时间。

ScheduledExecutorService 为预定执行或重复执行任务而设计，可以预定任务在初始延迟后只运行一次，也可以预定一个任务周期性执行。

执行器还可以控制一组任务，如 invokeAny 方法提交一个任务集合，并返回最先完成的任务结果；invokeAll 返回 Future 对象列表。构建 ExecutorCompletionService 能够按完成顺序获取结果，没有已完成结果时阻塞。