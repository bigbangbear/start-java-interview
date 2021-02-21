### HashCode

> - [ ] hashCode的作用？hashCode相等的两个对象一定相等吗？equals呢？反过来相等吗？

主要：用于查找的便利性，比如： HashMap 中查找存储的位置

特点：equals 相同 hashCode 相同，重写 equal 方法时需要重写 hashCode 方法。反过来不成立。

### 同步类容器

> - [ ] 实现原理？

答：通过 `synchronized` 关键字实现的同步类容器，由于性能问题，不建议使用

* Vector
* Stack
* Hashtable
* Collections.synchronizedXXX()

### 并发类容器

> - [ ] 实现原理？

答：通过同步机制实现，允许同一时刻多个线程同时访问，提高吞吐量，保证线程安全

1. ConcurrentHashMap 【1.7 及以前】

   > - [ ] 数据结构与实现方式 

   答：分段锁

> - [ ] size() 的实现方法？

答：刚一开始不加锁，前后计算两次所有segment里面的数量大小和，两次结果相等，表明没有新的元素加入，计算的结果是正确的。如果不相等，就对每个segment加锁，再进行计算，返回结果并释放锁

2. ConcurrentHashMap 【1.8】

> - [ ] 数据结构

答：【CAS + synchronized 实现】

> - [ ] size() 的实现方法？

答：通过对 baseCount 和 counterCell 进行 CAS 计算，最终通过 baseCount 和 遍历 CounterCell 数组得出 size

> - [ ] get() 为什么不需要加锁？

答：通过 volatile 实现内存的可见性

3. CopyOnWriteArrayList

> - [ ] 实现原理与数据结构？

答：在变更数组时，先获取锁，然后拷贝新增一个数组。

### 集合

> - [ ] ArrayList 数据结构？特点？

数据结构：**数组**，优点：**查找**支持快速随机访问，缺点：新增/删除操作的时间复杂度与插入的位置相关

> - [ ] LinkedList 数据结构？特点？

数据结构：双向链表，优点：新增/删除操作时间复杂度不与位置相关，缺点：不支持随机访问

单向链表：节省资源，增加了操作的复杂性

双向链表：操作简便，但是消耗了资源

> - [ ] HashMap 数据结构？特点？

数据结构：hashtable + 链表/红黑树

特点：以 key-value 的形式存储数据，时间复杂度为 n(1)

> - [ ] TreeMap 的数据结构？特点？

数据结构：红黑树

特点：对 key 进行排序存储

> - [ ] LinkedHashMap 数据结构？特点？

数据结构：HashMap + 双向链表

特点：记录插入的顺序

### 线程池

> - [x] 为什么需要使用线程池？

答：1、节约线程创建与销毁的资源与时间开销  2、方便多线程管理  3、提高响应速度

> - [x] 线程池的核心参数

答：coreSize、maxSize、keepAliveTime、BlockQueue、RejectHandler

> - [ ] 常用的四种线程池

| 线程池                  | coreSize | maxSize | keeyAliveTime | BlockQueue          | RejectHandler |
| ----------------------- | -------- | ------- | ------------- | ------------------- | ------------- |
| newCachedThreadPool     | 0        | MAX     | 60s           | SynchronousQueue    | AbortPolicy   |
| newFixedThreadPool      | n        | n       | 0             | LinkedBlockingQueue | AbortPolicy   |
| newSingleThreadExecutor | 1        | 1       | 0             | LinkedBlockQueue    | AbortPolicy   |
| newScheduleThreadPool   | n        | MAX     | 0             | DelayWorkQueue      | AbortPolicy   |

SynchronousQueue：当某个线程插入数据之后将会阻塞直到别的线程移除数据

DelayWorkQueue：基于堆的数据结构，根据任务执行时间升序排列

> 应用场景

| 线程池                 | 场景                                                         |
| ---------------------- | ------------------------------------------------------------ |
| newFixedThreadPoll     | 更新用户会话列表中某个字段，涉及到数据库操作，通过多线程提高更新效率 |
| newScheduledThreadPool | 长连接，定期发送事件给客户端                                 |

> - [ ] 拒接策略有哪些？

AbortPolicy：抛出异常

DiscardPolicy：直接丢弃

CallerRunsPolicy：调用者线程执行

DiscardOldestPolicy：直接丢弃最老未处理的请求

> - [ ] 线程数设置多少比较合理？

* CPU 密集型：n
* IO 密集型：2n

> - [ ] 线程池任务调度机制?

* workerCount < corePoolSize：新增工作线程执行
* workerCount >= corePoolSize && 阻塞队列未满：添加任务到阻塞队列
* 阻塞队列已经满 && workerCount < maxPoolSize：新增工作线程并执行
* 执行拒绝策略

> - [ ] 线程池的状态

![图3 线程池生命周期](https://tva1.sinaimg.cn/large/008eGmZEly1gnsyas98g3j30og063t93.jpg)

RUNNING：运行，接受新的任务提交，处理阻塞队列中的任务

SHUTDOWN：关闭，不再接受新的任务提交，继续处理阻塞队列中的任务

STOP：停止，不再接受新的任务提交，不再处理阻塞队列中的任务，中断正在执行的任务

TIDYING：整理，任务都已经中止

TERMINATED：结束

### 锁

> - [ ] synchronized 是什么？

synchronized 是 Java 关键字，用来保证多线程环境下最多只有一个线程访问临界区

> - [ ] synchronized 可以作用在哪里？

对象，实例方法

类对象，类方法

> - [ ] synchronized 实现原理？

在编译期间，在临界区前后插入 monitorenter 与 monitorexit 指令实现。本质上通过对象监视器( Monitor ) 进行获取实现，获取过程具有排他性，每一个时刻只有一个线程可以执行。没有获取到的线程会进入同步队列，等其他线程退出的时候重新获取 Monitor。

![Screen Shot 2021-01-12 at 19.35.02](https://tva1.sinaimg.cn/large/008eGmZEly1gml561wbf8j329j0u0ng2.jpg)

monitorenter：

* 如果 monitor 计数器为 0，当前线程立刻获取锁，计数器 + 1
* 如果 monitor 计数器大于 0，而且是当前线程获取锁，计数器 +1
* 如果 monitor 计数器 大于 0，而且不是当前线程，则当前线程获取失败，进入同步队列

> - [ ] synchronized 为什么性能较差，需要进行锁优化？

由于 monitorenter 与 monitorexit 操作依赖于操作系统的互斥锁 (mutex lock) 实现，由于使用Mutex Lock 挂起线程和恢复线程的操作都需要转入内核态中完成，耗费的资源较大

> - [ ] synchronized 锁优化 [参考1](https://www.pdai.tech/md/java/thread/java-thread-x-key-synchronized.html) [参考2](https://blog.csdn.net/qq_27093465/article/details/106122411)

无锁：

偏向锁：

* 背景：研究发现大多数情况下不存在锁的竞争，而总是同一线程多次获取锁。
* 解决：通过在 Mark Word 存储线程的 ID 进行比较，如果与当前线程 ID 相同执行同步方法。如果 ID 不同，通过 CAS 竞争锁，如果竞争失败升级为轻量级锁

轻量级锁：

* 背景：为了在线程近乎交替执行同步块时提高性能
* 解决：先在当前线程的栈帧建立 LockRecord (锁记录)空间。(1) 拷贝对象的 MARK WORD 到锁记录中。(2) 通过 CAS 尝试更新对象的 MARK WORD 指向锁记录指针。(3) 更新成功，线程获取到锁。(4) 更新失败，首先先自旋尝试获取锁。多次失败后升级为重量级锁

重量级锁：通过监视器锁来实现

![img](https://img-blog.csdnimg.cn/20200514162040282.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MDkzNDY1,size_16,color_FFFFFF,t_70)

> - [ ] 谈谈ThreadLocal的作用？底层如何实现的？会不会导致内存泄露？

作用：每个线程拥有自己的专属变量

实现：每个线程具有一个 Map，键为 ThreadLocal 对象，值为对应的值。存储的时候，先获取 Map，然后调用 Put 方法进行设置。调用的时候，获取线程的 Map，已 ThreadLocal 为键进行获取

内存泄漏：Map 的键通过弱引用解决内存泄漏。但是 Value 可能存在内存泄漏，只有当引用的线程结束才会释放。

解决方案：在不使用 ThreadLocal 时调用 Remove 函数

使用场景：在我们的应用中，有个参数掉调用链上的多个方法都需要使用，就存放在 ThreadLocal 中，避免逐层透传

> - [ ] 谈谈volatile的作用？实现原理以及使用场景[参考](https://www.cnblogs.com/dolphin0520/p/3920373.html)

作用：内存的可见性、防止指令重排

实现原理：通过内存屏蔽实现，保证之前的所有指令都执行后再执行之后的指令。修改理解写入主存，并且使得其他线程的缓存失效，从内存中重新获取值。

> - [ ] 了解 Happen-Before 原则吗？

作用：多线程共享变量的可见性

1. 锁
2. volatile
3. 线程启动
4. 线程终止
5. 线程中断
6. 终结器
7. 传递性

> - [ ] 线程的状态？[参考](https://www.pdai.tech/md/java/thread/java-thread-x-thread-basic.html)

![image](https://www.pdai.tech/_images/pics/ace830df-9919-48ca-91b5-60b193f593d2.png)

> - [ ] 线程常用函数的作用

`Thread.sleep(millis)`：当前线程进入阻塞状态，但不释放对象锁，时间到后自动进入可运行状态;

`Thread.yield()`：当前线程调用此方法，当前线程放弃获取cpu时间片，由运行状态变为可运行状态;

`t.join()/t.join(millis)`：当前线程调用其它线程t的join方法，当前线程阻塞，但不释放对象锁，直到线程t执行完毕或者millis时间到，当前线程进入可运行状态。

`obj.wait()/obj.wait(millis)`：当前线程释放对象锁，进入等待队列。依靠notify()/notifyAll()唤醒或者millis时间到自动唤醒。

`notify()/notifyAll()`：随机唤醒一个/所有等待该对象锁的线程，进入就绪队列等待CPU的调度；

> - [ ] AQS (abstract queue synchronizer) 的作用

AQS：是用来构建锁与同步器的框架 (是什么)，底层通过 CAS 修改共享变量 state 实现互斥访问，同时利用 FIFO 队列实现线程间的阻塞排队与唤醒 (怎么做)，ReentranceLock、CountDownLatch 都是基于 AQS 实现的

> - [ ] ReentrantLock  的作用？实现

作用：JDK 实现的可重入锁 (是什么)，避免同一线程重复获取锁造成死锁。

> - [ ] ReentrantReadWriteLock 的作用？

作用：可重入读写锁，提高并发读的性能，适用于读多、写少的场景。

> - [ ] Semaphore 的作用？实现

作用：信号量，允许多个线程同时访问共享资源。

> - [ ] CountDownLatch 的作用？

作用：将一个任务划分为 n 个子任务，子任务结束后调用 countDown，等待任务执行的线程调用 `await()` 等待所有子任务结束

> - [ ] 介绍一个使用到锁与同步器的使用场景

场景：同步会话数据到 ES 中，由于有不同的企业，并且会话数据量较大，为了提高性能。利用线程池开启多个线程处理同步任务。通过 CountDownLatch 等待线程池执行完成，利用 Semaphore 限制同步任务的数量，避免对 ES 造成太大的压力