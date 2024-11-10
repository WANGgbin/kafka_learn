描述 kafka 中事务相关实现。<br>

# 为什么要有事务

在涉及同时操作多个主题/分区的场景下，比如同时写多个主题，我们就需要保证要么同时多个主题都写入成功，要么都没有写入。否则，会出现数据一致性问题。

当然，不使用事务，我们也可以通过后台任务重试的方式来解决都生产成功的问题，但这种方式有以下两个问题:

- 代码逻辑变得复杂，不好维护
- 即使不断重试，仍然有可能会失败


而使用事务后，业务逻辑就会简化很多，易于后续的维护。


# 实现原理

## transaction coordinator

问题：

- 在提交/回滚的时候，如果某个节点失败了，怎么办？一直重试吗？


与消费者组协调器类似，kafka 通过 transaction coordinator 来保存事务相关消息，充当分布式事务中的协调者角色。

与 __consumer_topic 类似，kafka 引入了 _transaction_state 来保存事务消息，每一个 producer 对应一个分区，分区的分配方式也是通过 hash + mod 方式确定。

producer 通过以下的调用来使用事务：

- init_transaction
- 开启一个循环
  - begin transaction
  - commin/abort transaction

### init transaction

#### 确定 __transaction_state partiton leader

首先，producer 需要确定与自己对应的 __transaction_state 分区 leader broker 是哪一个？如何确定的呢？

producer 给集群中某个节点(负载最低节点，即目前 inflightRequests 最少的节点)发送 `FindCoordinatorRequest` 请求，broker 通过分配算法确定关联的 __transaction_topic 分区，

并将分区的 leader 节点信息通过 `FindCoordinatorResponse` 的方式发送给 producer。

#### 初始化事务信息

在确定了 coordinator 后，producer 会给 coordinator 发送 `InitProduceIdRequest`， coordinator 会给 producer 分配 producerID、producerEpoch 并创建事务相关的元信息并持久化

到 __transaction_state 中。


那么 coordinator 是如何分配 produceID 的呢？通过 `zookeeper` 完成的，zookeeper 有一个特定的节点，存储当前已分配的最新的 producerID，同时 zookeeper 每个节点都自带版本信息。

coordinator 就是通过乐观锁的方式分配 producerID 的。源码如下：

```scala
object ZkProducerIdManager {
  def getNewProducerIdBlock(brokerId: Int, zkClient: KafkaZkClient, logger: Logging): ProducerIdsBlock = {
    // Get or create the existing PID block from ZK and attempt to update it. We retry in a loop here since other
    // brokers may be generating PID blocks during a rolling upgrade
    var zkWriteComplete = false
    while (!zkWriteComplete) {
      // 从 zookeeper 获取已分配的 id 和节点的版本信息
      val (dataOpt, zkVersion) = zkClient.getDataAndVersion(ProducerIdBlockZNode.path)

      // generate the new producerId block
      val newProducerIdBlock = dataOpt match {
        // 如果节点已经存在
        case Some(data) =>
          val currProducerIdBlock = ProducerIdBlockZNode.parseProducerIdBlockData(data)

          // 基于当前 producerID 分配 1000 个 id
          new ProducerIdsBlock(brokerId, currProducerIdBlock.nextBlockFirstId(), ProducerIdsBlock.PRODUCER_ID_BLOCK_SIZE)
        case None =>
          // 节点不存在，则创建，初始值为 0
          new ProducerIdsBlock(brokerId, 0L, ProducerIdsBlock.PRODUCER_ID_BLOCK_SIZE)
      }

      val newProducerIdBlockData = ProducerIdBlockZNode.generateProducerIdBlockJson(newProducerIdBlock)

      // 更新 zookeeper，这里是通过乐观锁的方式更新的。如果发生了冲突，则再次尝试更新
      val (succeeded, version) = zkClient.conditionalUpdatePath(ProducerIdBlockZNode.path, newProducerIdBlockData, zkVersion, None)
      zkWriteComplete = succeeded

      // 更新成功直接返回
      if (zkWriteComplete) {
        return newProducerIdBlock
      }
    }
  }
}
```
### begin transaction

producer 只是更改本地事务状态，这一步不会与 coordinator 交互。


### 发送数据

在给实际的分区 leader 发送消息前，还需要给 coordiantor 发送分区信息。因为，kafka 事务是个分布式事务，需要协调多个 broker，因此 coordinator 就需要知道最后到底要协调哪几个 broker，凡是

发送了数据 or 对应的 consumer group coordinator partition leader，coordinator 都需要感知。


在给 coordinator 发送了分区信息后，就可以给分区 leader 发送消息了，发送流程跟普通消息的发送流程没有区别。

这里有个问题，发送之后，消费者不就可以看到这个消息了？有可能。我们在消费者部分描述事务场景下的消费逻辑。


### 提交消费位移

提交位移整个过程跟给普通 partition leader 发送事务消息类似，也分为两步：

- coordinator 通过 groupID 计算 __consumer_offset 分区信息，并保存到事务元信息中
- producer 给 __consumer_offset partition leader 提交位移信息，当然这个位移信息只有事务提交才生效


### 事务 commit/abort

我们以 commit 举例，producer 给 coordinator 发送事务提交请求，coordinator 修改事务状态为 prepareCommit，然后给保存的所有分区的 leader 发送提交事务请求，各个 leader 接收到请求后会在日志中写入一个<br>
control batch(该 batch 中只有一条 record)。当分区 ISR 所有副本都同步了该控制消息后，分区 leader 给 coordinator 发送响应。<br>

当 coordinator 接收到所有分区 leader 的响应后，就会标记事务为结束，然后删除事务对应的元信息。

## LSO 管理

普通分区 leader 也为事务维护了一个上下文，包括 ongoingTxns(进行中的事务)、unreplicatedTxns(leader 已经写入事务 control record，但是 ISR 中副本尚未同步该记录)，两者都是 map 结构，key 为事务第一条消息的 offset，value 为事务信息。当 ISR 中所有副本都同步了该控制记录后，leader 就会把该事务移除并尝试更新 LSO。当然，具体实现上，与 update HW 逻辑类似，在收到 replicate 的 fetch 请求的时候，就会尝试更新 LSO，LSO 取值为 ongoingTxns 与 unreplicatedTxns 中最小的偏移量。其源码如下：

```java
/**
     * An unstable offset is one which is either undecided (i.e. its ultimate outcome is not yet known),
     * or one that is decided, but may not have been replicated (i.e. any transaction which has a COMMIT/ABORT
     * marker written at a higher offset than the current high watermark).
     */
    public Optional<LogOffsetMetadata> firstUnstableOffset() {
        Optional<LogOffsetMetadata> unreplicatedFirstOffset = Optional.ofNullable(unreplicatedTxns.firstEntry()).map(e -> e.getValue().firstOffset);
        Optional<LogOffsetMetadata> undecidedFirstOffset = Optional.ofNullable(ongoingTxns.firstEntry()).map(e -> e.getValue().firstOffset);

        if (!unreplicatedFirstOffset.isPresent())
            return undecidedFirstOffset;
        else if (!undecidedFirstOffset.isPresent())
            return unreplicatedFirstOffset;
        else if (undecidedFirstOffset.get().messageOffset < unreplicatedFirstOffset.get().messageOffset)
            return undecidedFirstOffset;
        else
            return unreplicatedFirstOffset;
    }
```

## consumer

consumer 能消费哪些事务消息呢？这就涉及到 **隔离级别** 的概念。在 kafka 中，针对于事务消息，消费者有两种隔离级别：READ_COMMITED、READ_UNCOMMITED。

#### READ_COMMITED

READ_COMMITED，即只有消息提交后，消费者才可以看到消息。那么是怎么实现的呢？

我们知道对于普通信息，consumer 只能读取到 HW(High Watermark) 之前的消息。在引入 kafka 事务后，多了一个 LSO(Last Stable Offset) 的概念，LSO 之前所有的事务消息都是提交的。<br>
在 READ_COMMITED 隔离级别下，consumer 只能读取 min(LSO，HW) 之前的消息。

现在有个问题，因为事务的控制消息是在普通事务消息之后写入的，那么消费者在读取到一条事务消息的时候，怎么知道对应的事务提交还是回滚了呢？一种思路是可以先不处理该消息而是暂存下来，只有<br>
读取到对应的控制消息后，再根据控制消息类型决定是否消息前面暂存的所有事务消息。但这种方式打破了 **顺序消费** 的保障，消费者不再按照顺序的方式消费消息。

实际上，消费者从 broker fetch 的 resp 中，除了 records 外，还包括 abortedTxns 信息，该信息含义为本次 fetch 的 事务 records 对应的 abortedTxn 信息。这样消费者在遇到事务消息的时候，就可以判断其 producerID<br>
是不是在 abortedTxns 中，如果在就表示事务 aborted 了，直接忽略当前记录，否则就表示事务提交了，消费当前记录。对应的源码如下：

```java
    // 获取下一条记录
    private Record nextFetchedRecord(FetchConfig fetchConfig) {
        while (true) {
            // 当前 batch 中 record 处理完毕，获取下一个 batch
            if (records == null || !records.hasNext()) {
                currentBatch = batches.next();
                
                // 注意只有 READ_COMMITTED 隔离级别下，才需要判断事务消息是不是 abort
                // 事务消息与普通消息的区别：事务消息包含 producerId
                if (fetchConfig.isolationLevel == IsolationLevel.READ_COMMITTED && currentBatch.hasProducerId()) {
                    // remove from the aborted transaction queue all aborted transactions which have begun
                    // before the current batch's last offset and add the associated producerIds to the
                    // aborted producer set
                    consumeAbortedTransactionsUpTo(currentBatch.lastOffset());

                    long producerId = currentBatch.producerId();
                  //  当前事务消息 aborted 了，直接跳过
                  if (isBatchAborted(currentBatch)) {
                        nextFetchOffset = currentBatch.nextOffset();
                        continue;
                    }
                }

                records = currentBatch.streamingIterator(decompressionBufferSupplier);
            } else {
                // 消费 batch 中的 record
                Record record = records.next();
                // skip any records out of range
                if (record.offset() >= nextFetchOffset) {
                    // 如果是控制消息，则直接忽略
                    if (!currentBatch.isControlBatch()) {
                        return record;
                    } else {
                        // Increment the next fetch offset when we skip a control batch.
                        nextFetchOffset = record.offset() + 1;
                    }
                }
            }
        }
    }
```

我们接着看看判断当前事务消息是否 aborted 的逻辑：

```java
private boolean isBatchAborted(RecordBatch batch) {
        return batch.isTransactional() && abortedProducerIds.contains(batch.producerId());
    }
````

这个 abortedProducerIds 就是根据 fetch response 中的 abortedTxns 创建的。

```java

// 设置  abortedTransactions 字段
private PriorityQueue<FetchResponseData.AbortedTransaction> abortedTransactions(FetchResponseData.PartitionData partition) {
        if (partition.abortedTransactions() == null || partition.abortedTransactions().isEmpty())
            return null;

        // 注意看，这里是个优先级队列，队列根据每个事务的首条消息偏移量排序。
        PriorityQueue<FetchResponseData.AbortedTransaction> abortedTransactions = new PriorityQueue<>(
                partition.abortedTransactions().size(), Comparator.comparingLong(FetchResponseData.AbortedTransaction::firstOffset)
        );
        abortedTransactions.addAll(partition.abortedTransactions());
        return abortedTransactions;
    }

// 在消费事务消息的时候，会将所有首条消息偏移量小于当前偏移量的事务的 producerIDs 加入到 abortedProducerIds 中。
// 这是显而易见的，当前消息对应的事务的起始位移一定小于等于当前偏移量。 
private void consumeAbortedTransactionsUpTo(long offset) {
        if (abortedTransactions == null)
            return;

        while (!abortedTransactions.isEmpty() && abortedTransactions.peek().firstOffset() <= offset) {
            FetchResponseData.AbortedTransaction abortedTransaction = abortedTransactions.poll();
            abortedProducerIds.add(abortedTransaction.producerId());
        }
    }
```

那么在 broker 中，是如何维护 abortedTxn 的呢？

这部分信息是需要持久化的，在 kafka 的 log 目录中，每个 logsegment 对应一个 txnindex 文件，该文件保存对应的 log segment 中 aborted 的事务元信息，每个事务包括：firsetOffset、lastOffset、LSO(当前时刻 LSO 取值)。<br>
为什么要存储 LSO 呢？这跟 broker 处理 fetch 请求时，查找一定 offset 范围内的 abortedTxn 相关。我们看看代码实现：

```scala
  // segment 为 startOffset 对应的 segment
    private def collectAbortedTransactions(startOffset: Long, upperBoundOffset: Long,
                                         startingSegment: LogSegment,
                                         accumulator: Seq[AbortedTxn] => Unit): Unit = {
    val higherSegments = segments.higherSegments(startingSegment.baseOffset).iterator
    var segmentEntryOpt = Option(startingSegment)
    // 要获取 [startOffset, upperBoundOffset] 中所有事务消息对应的 abortedTxns，可能需要遍历多个 segment 的 txnIndex 文件
    while (segmentEntryOpt.isDefined) {
      val segment = segmentEntryOpt.get
      val searchResult = segment.collectAbortedTxns(startOffset, upperBoundOffset)
      accumulator(searchResult.abortedTransactions.asScala)
      if (searchResult.isComplete)
        return
      segmentEntryOpt = nextOption(higherSegments)
    }
  }

// 从 segment 中收集 abortedTxns 逻辑
public TxnIndexSearchResult collectAbortedTxns(long fetchOffset, long upperBoundOffset) {
        List<AbortedTxn> abortedTransactions = new ArrayList<>();
        for (AbortedTxnWithPosition txnWithPosition : iterable()) {
            AbortedTxn abortedTxn = txnWithPosition.txn;
            // 只有事务范围跟请求范围有 overlap，就是需要的 abortedTxns
            if (abortedTxn.lastOffset() >= fetchOffset && abortedTxn.firstOffset() < upperBoundOffset)
                abortedTransactions.add(abortedTxn);

            // 这就是 txnIndex 中每个 txnEntry 为什么要存储 LSO 的原因。
            // 标识 upperBoundOffset 之前消息在生成此 abortedTxn 时，已经提交了，那么其对应的 abortedTxn 消息一定在当前 abortedTxn 之前，那么就不需要继续遍历 txnIndex 文件了。
            // result 的第二个参数表示是否继续遍历 txnIndex 文件。
            if (abortedTxn.lastStableOffset() >= upperBoundOffset)
                return new TxnIndexSearchResult(abortedTransactions, true);
        }
        return new TxnIndexSearchResult(abortedTransactions, false);
    }
```
#### READ_UNCOMMITED

即使消息没有提交，消费者也可以直接消费。


