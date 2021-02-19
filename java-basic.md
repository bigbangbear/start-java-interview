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

