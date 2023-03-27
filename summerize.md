# kafka 是什么?
kafka 是一款开源的分布式消息中间件. 具有高性能,高可用,持久化存储等特性.

# 使用场景
- 削峰添谷
- 服务解偶
- 异步处理
- 日志收集

如果要收集并分析应用的日志,应用可以通过异步的方式将消息发送到消息队列中,通过消费这些日志消息,进行日志的处理.
- 用户行为数据收集

广告公司经常需要评估某个广告的 CTR, CVR 这些数据. 当用户操作了某个广告后, 可以发送一条信息到消息队列中, 通过消费这些消息来统计后验数据.

# kafka 的架构什么样?都包含哪些角色?
- producer

  生产者

- broker

  kafka 集群中的每一个实例都是 broker, 用于消息的持久化存储. 每个 broker 除了处理消息外,还可能担任其他的角色.
  - controller

    控制器,一个 kafka 集群中只有一个 控制器. 控制器用于集群中所有的主题和分区的管理.
  - transaction coordinator

    当生产者是有事务的时候,会存在一个与生产这匹配的事物协调器.
  - group coordinator

    每个消费者组都会对应一个组协调器,主要用于消费者组的 rebalance.

- consumer

  消费者,负责消费某个 topic 的消息

- topic

  topic 是一类消息

- partition

  每一个 topic 下可以有多个分区, 多分区机制加快生产者和消费者的速度. kafka 保证没一个分区的写是顺序的(严格来说,并不总是这样,在介绍生产者时候会介绍). 一个分区只能被一个消费者实例消费, **因此,如果消费组中消费者实例的数量超过 topic 分区的数量,多余的消费者是空闲的**.

- replication

  副本. 实际上,每一个分区可以有多个副本,正是多副本机制保证了 kafka 的高可用. 当 leader 副本所在的实例宕机后, controller 会使用一定的算法,从分区的其他副本中重新选出 leader 副本.


# 总结

B 站有关于 kafka 很好的总结, 参考:
- [我用 kafka 两年, 踩坑无数](https://www.bilibili.com/video/BV1gm4y1w7tN/?spm_id_from=333.999.0.0)
- [MQ 中六个坑](https://www.bilibili.com/video/BV1yv4y117Kj/?spm_id_from=333.999.0.0)