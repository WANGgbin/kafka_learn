介绍 kafka 中日志的格式.

# 消息格式

总共有三个版本的消息格式, 我们只关注 v2 版本. v2 版本的消息是以 record_batch 的形式存在的. record_batch 包括头部  以及  records. record_batch 对应生产者端的 produce_batch.

那么 v2 版本有哪些优化呢?
- 消息更小

通过引入 varint 以及 相对位移/时间戳(每个 record 只需记录相对于第一条消息的 相对位移, 相对时间戳,减少不必要的空间浪费), 大大减小了包的长度.

- 支持幂等/事务

record_batch 的 header 中引入了 produce_id, produce_epoch, first sequence 等字段.

# 日志目录布局

每个分区对应一个文件夹, 文件夹中包括若干日志文件. 因为当消息很多的时候, 文件会进行分段. 进行分段的一个很重要的目的是为了加快消息的查找. 每个日志文件对应一个偏移量索引文件和一个时间戳索引文件. 文件夹的命名形式为: **<topic>-<partition>**, 日志文件和两个索引文件都是根据基准偏移量命名的, 名称固定为 20 位数字. 比如如果偏移量为0, 则文件命名为:
00000000000000000000.log
00000000000000000000.index
00000000000000000000.timeindex

若干重要的参数:
- log.segment.bytes

当日志文件的大小超过此参数设置的值后, 文件进行分段. 默认值为 1GB.

- log.index.size.max.bytes

表示索引文件最大的大小, 默认值为 10MB. 当索引文件大小超过此值的时候, 日志文件也会进行分段.

- log.index.interval.bytes

每当日志文件中写入该参数代表的大小的消息后, 便在索引文件中增加一项. 该参数的默认值为 4KB

# 索引文件
为了加快消息的检索, kafka 中的索引文件是通过 **mapping 的方式直接映射到进程内存空间中的**.

- 偏移量索引文件

我们都知道消费者可以根据特定的偏移量从 broker 拉取特定的消息, 这是怎么实现的呢? 这就是偏移量索引文件的作用. 

索引文件中的每一项格式为: | relative_offset | position |
之所以采用 relative_offset 的方式, 也是为了节省内存空间. 

那么根据偏移量查询消息的流程是什么样的呢?
  - 确定消息所在的日志分段

    kafka 采用跳表的数据结果保存每个日志文件中各个日志分段的 base_offset, 我们可以通过这个跳表快速定位到对应的日志分段.

  - 根据索引文件进行二分查找

    可以通过索引文件快速查找到最后一个小于 target_offset 的消息在文件中的 position
  
  - 遍历
    从 positon 开始遍历每条消息, 直到查到目标消息.

- 时间戳索引文件

时间戳索引文件与偏移量索引文件类似,不再描述.

# 日志清理

当 broker 中的日志越来越多的时候怎么办呢?  这就涉及到日志的清理策略. kafka 中包括两种日志清理方式:
- 日志删除

broker 内部会有一个定时删除日志的任务, 这个周期可以通过`log.retention.check.interval.ms` 来指定, 默认值为 5min.

日志删除有三种策略:
  - 基于时间

    可以通过参数 `log.retention.hours`, `log.retention.minutes`, `log.retention.ms` 来设置时间. 默认只设置了 log.retention.hours, 默认值为 7d. 如果多个参数同时设置了, log.retention.ms 的优先级最大.

    当某条消息的时间戳 < current_time - log.retention.hours 的时候, 就会被删除.
    删除流程为, 先将内存中待删除日志段对应的跳表中的结构删除,这样就不会有线程对这些日志分段进行读操作. 然后再物理删除日志分段. 后面的基于大小,基于日志偏移量的策略,在确定了待删除日志分段后,删除的策略都是一样的.

  - 基于大小

    通过参数 `log.retention.bytes` 来指定日志的大小, 当日志超过这个值的时候, 就会进行删除. 该参数的默认值为 -1, 即无穷大.

  - 基于消息偏移量

    当日志分段的下一个日志的 base_offset 小于 log_start_offset 的时候, 该日志分段就会被删除. log_start_offset 默认就是第一个分段的 base_offset. 但是我们可以通过 kafka脚本或者 KafkaAdminClient 的 deleteRecords 方法来发送 DeleteRecordsRequest 删除消息, DeleteRecordsRequest 的本质仅仅是重置 log_start_offset.

- 日志压缩

日志压缩即 `log compaction`. 对于具有相同 key 的不同 value, 日志压缩会保留最新版本的消息.因此 如果应用想要保留相同 key 最新的消息, 就可以使用 日志压缩 方式来清理日志. kafka 的 内部主题 __consumer_offsets, __transaction_state 都采用的是日志压缩.

**需要特别注意: 因为 log compaction 是根据消息的 key 进行日志清理的, 因此应用一定要保证消息的 key 部位 null!!!**

三个问题:
- 什么时候会触发 log compaction?

为了避免不必要的 log compaction. kafka 引入了污浊率的概念. 即分区对应的日志中, 未清理的部分占当前文件的比例. 当超过参数`log.cleaner.min.cleanable.ratio`设定的值的时候(默认值为 0.5),就会触发 log compaction.

- log compaction 的流程是什么样的呢?

因为是要保留 key 最新的消息, 线程内部肯定会保留以消息 key 为 key 的哈系表. 但是 value 应该是什么呢? 如果是消息的 value 的话, 没有那个多内存可用. 我们可以使用消息的 offset 作为 value, 因为我们可以通过 offset 定位到对应的消息. 除此之外, key 是以 md5 之后的结果保留的. 因此内存中的哈系表很小的空间就可以处理很大的日志段.

第一次遍历日志(**offset 从 0 到 active segment 的 baseoffset**), build 内存中的哈系表. 第二次遍历日志, 根据哈系表重建日志下的日志段. 因为重建后的日志段文件肯定会变小,为了避免太多的小文件. 在每次 log compaction 之前将若个日志段划分到一个组中. 划分规则就是根据日志段文件大小(<=1G)以及索引文件大小(<=10M). 在重建日志段的时候, 每个日志分组就会对应一个新的日志段.

- log compaction 虽然只保留每个 key 最新的 value, 但是随着 key 越来越多, log 也越来越大, 怎么删除消息呢?

通过添加一个墓碑消息(key 部位 null, value 为 null 的消息), 来表示删除 key 对应的消息.
# 零拷贝