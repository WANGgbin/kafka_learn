kafka 的事务指的是什么呢?

# exactly once sematic
在介绍事务之前, 我们先来聊聊 kafka 中的 EOS 语义, 即 精确一次语义.
一般而言, 消息队列的消息保障有三个级别, 分别为:
- at most once 最多一次
- at least once 至少一次
- exactly once 有且只有一次

当我们通过 kafka 发送消息的时候, 如果发送消息失败(此时, 消息可能已经成功写入到 broker 中), 我们会进行重试, 此时可能会导致消息的重复, 这种情况下对应的保障级别就是 at least once.

kafka 从 0.11.0 开始引入了 **幂等** 和 **事务** 的概念来保证 **生产端** 的 EOS 语义.

# 幂等
kafka 中的幂等是站在生产者角度而言的, 指的是同样的一条消息, 在 brokder 端 只会被存储一次.

通常要实现幂等, 请求方需要携带一个唯一的标识, 服务方需要存储之前请求的标识, 通过判断请求的标识符有没有在已经请求的标识符集合中来判断是否真正处理请求.

当生产者开启幂等特性后, 在发送数据的时候, kafka 会给生产者分配一个唯一的表示 produceID, 生产者会给每个分区分别维护一个从 0 开始的序列号(sequence number), 每条消息一个序号.

注意, 幂等是保证消息的顺序投递的, 但是参数 `max.in.flight.requests.per.connection` 却可以大于 1. 这是怎么回事呢?

在服务端, broker 给分区的每个生产者维护了一个长度为 5  的队列, 队列中的元素是 produceBatch 的元信息, 主要包括, 最后一条信息的 sequence number, batch 的消息数. 队列中的元素按照 sequence number 严格排序.

当 broker 接受到一条新的 produceBatch 的时候, 首先会判断当前队列中是否已经存在跟此 produceBatch 匹配的元素, 如果有,说明该 produceBatch 已经发送过了. 如果没有则判断当前 produceBatch 第一条消息的 sequence number 是否等于队列中最后一个元素的最后一条信息的 sequence number + 1, 如果是则说明当前是有序的, 会将该 produceBatch 的元信息存储到队列中,并持久化该 produceBatch. 如果不是, 则说明是乱序的, 前面的消息可能丢失了, 此时会发送 `OutofOrder` 异常.

当生产者接受到该异常后, 会进行重试. 但是如果前面有消息还没有发送到 broker, 此时的重试是毫无意义的, 因此生产者在重试的时候, 会判断当前的 produceBatch 是不是 in flight 的第一个 produceBatch, 即它之前的消息都已经成功发送. 若是则继续重试.

我们说生产者在设置 `max.in.flight.requests.per.connection` 的时候, 不能大于 5 为什么呢? 这实际上是跟 broker 端维护的队列长度有关系. 如果长度大于 5, 假设发送了 1, 2, 3, 4, 5, 6 总共 6 条消息, 都成功发送到了 broker 端, 6 对应的元信息会覆盖 1 对应的元信息, 但是如果生产者没有收到 1 对应的 ack, 则会进行重试, 但此时 broker 端已经没有 1 对应的元信息, 从而抛出 `OutofOrder` 异常, 生产者会不断重试, 直到重试次数到达上限. 相反, 如果 `max.in.flight.requests.per.connection` 小于等于 5, 当 broker 队列中 因为追加新的元素,导致旧的元素被移除队列时, 我们可以确定旧的元素的 ack 已经被客户端收到, 如果没有收到, 那么相应的 produceBatch 还在 in flight 队列中, 那么就不可能导致 broker 队列满.


为什么不直接把 `max.in.flight.requests.per.connection` 设置为 1 呢? 这不就保证了消息的顺序性吗? 可以这么做, 但是这会大大的限制 kafka 的吞吐. 实际上, 设置该参数 大于 1, 发送乱序也是极少数的情况, 只要能正确应对这些异常情况就可以了.

注意:
- 当生产者异常时, 恢复正常后, 会被分配一个新的 produceID, 因此 kafka 的幂等是不跨会话的.
- 生产者会分别给每个分区分配从 0 开始的序列号, 因此 kafka 的幂等仅仅是相对于某个分区而言的.
- 如果我们手动发送两条完全相同的消息, 站在生产者的角度看, 这两条消息会被赋予不同的 sequence number.

那么如果解决跨分区, 跨会话的 EOS 语义呢? 这就是 kafka 事务.

# 事务
kafka 中的一个典型场景是:
- 从某个 topic 消费某些消息
- 逻辑处理
- 发送到另一个 topic

通常我们希望上述操作是一个原子操作, 这就是 kafka 事务保证的. 我们以上述场景出发, 看看 kafka 是如何实现事务的.

首先, 我们需要了解事务协调器这个概念. 每个生产者会唯一对应一个事务协调器, 事务协调器用来管理事务相关的操作. 那么如何确定生产者对应的事务协调器是哪个呢? 

当生产者要开启事务的时候, 需要指定 transactionId 这个属性, 即 事务 id. 跟 __consumer_offsets 一样, kafka 内部还存在一个跟事务相关的内部主题 `__transaction_state`. 之后跟通过 consumerGroupId 选择组协调器一样, 我们可以通过 事务id 找到对应的事务协调器.

kafka 提供了 5 个事务相关的 api:
- void initTransactions()

此阶段会尝试寻找对应的事务协调器 并发送一个获取 produceID 的请求, 事务协调器查询内存中数据结构判断是否是第一次给该事务 id 分配 produceID, 如果是通过 zookeeper 分配一个 produceID, 并将 transactionID 和 produceID 的对应关系更新到内存的数据结果中并持久化到 __transaction_state 中. 如果不是第一次分配, 则直接从内存中获取对应的 produceID 并更新对应的 epoch(+1, 并持久化到磁盘中).

注意, 如果不是第一次分配, 同时会判断之前事务的状态, 如果未完成, 会进行提交或者中止.

- void beginTransaction()

本地标记开启一个事务.

- 发送消息

当生产者给一个新的分区发送消息的时候, 首先会给事务协调器发送一个消息, 事务协调器会将 <trsactionID, topicPartition> 的映射关系更新到内存并持久化到磁盘中. 保存了这些分区信息, 后续才知道往哪些分区发送 COMMIT 或者 ABORT 控制信息.

- void sendOffsetsToTransaction(map<TopicPatition, OffsetAndMetadata> offsets, string consumerGroupId);

当开启事务后, 我们需要关闭位移的自动提交, 同时也不能手动提交位移. 而是通过发送位移信息给事务协调器, 事务协调器会在对应的事务中记录 consumerGroupId 对应的 __consumer_offsets 分区信息, 并发送位移信息给对应的组协调器.

但是, 特别注意, 此时虽然位移消息发送给了组协调器, 但是此位移信息并没有生效.
- void commitTransaction();

当调用此函数时, 事务协调器收到请求后, 会给消息分区 和 __consumer_offsets 分区发送 commit 控制信息. 此时, 提交给组协调器的位移信息才生效.

客户端是怎么消费事务消息的呢? 客户端有个参数 `isolation_level` 可以设置事务隔离级别. 隔离级别只有两个值: `read_uncommited` 和 `read_commited`, 默认值是 read_uncommited. 

在 read_uncommited 隔离级别下, 消费者正常消费事务消息. 在 read_commited 级别下, 消费者并不会直接消费事务消息, 而是先缓存, 等接受到控制信息的时候, 要么投递给消费者(COMMIT), 要么丢弃消息(ABORT).
- void abortTransaction();

基本同 commitTransaction(), 只不过发送的控制消息是 ABORT.
