### 1. MQ 是什么？有什么优点

MQ：消息队列，

优点：解耦、削峰、异步

缺点：系统的复杂性、需要考虑高可用、数据丢失、数据重发、有序发送等问题

### 2. 常用消息队列比较

| 对比     | Active MQ | Rabbit MQ | Rocket MQ | Kafka      |
| -------- | --------- | --------- | --------- | ---------- |
| 吞吐量   | 1W        | 1W        | 10W       | 10W        |
| 高可用   | 主从      | 主从      | 分布式    | 分布式     |
| 时延     | 1ms       | 1ns       | 1ms       | 1ms        |
| 维护     | Java      | Erlang    | java      | java/scala |
| 顺序消息 | 不支持    | 不支持    | 支持      | 支持       |
| 持久化   | 少量堆积  | 少量堆积  | 大量堆积  | 大量堆积   |

### 3. 消息队列顺序问题 [参考](https://sa.sogou.com/sgsearch/sgs_tc_news.php?req=nxiYB_RqrtR_RCBzn7Mur64FYacCUx0frbgmBhIDEUQ=&user_type=1)

> 生产者生产的消息需要被消费者按照顺序有序消费

Kafka：在相同 Patition 上可以保证消息消费的有序性，根据规则将消息转发到相同 Broker 的 Patition 上，从而保证消息消费的有序性。

> 消费者多个线程并发处理导致消息消费乱序

在消费者中维护多个内存队列，通过将 Key Hash 到指定的队列上，让相同的线程处理对应队列的数据，从而避免乱序消费问题

