我们知道 broker 在保存日志的时候，如果使用同步刷盘的方式，性能就会很差。但如果不同步刷盘，就可能会导致数据的丢失。<br>
在 kafka 中，通过 ISR 多副本方式保证数据不会丢失，即使 partition 的 leader 异常退出，也可以从 partition 的其他副本中选举一个新的 leader。<br>
这也是一个值得我们学习的地方：**通过多副本保存消息，而不是同步持久化到磁盘，这样性能更好，而且可以避免单机故障**

那么 broker 是什么时候将 log 持久化到磁盘的呢？<br>
实际上，broker 在启动的时候会拉起后台任务，该任务通过定时的方式将 log 持久化到磁盘中，本篇我们看看具体的持久化逻辑。


入口逻辑如下：

```scala
  /**
   * Flush any log which has exceeded its flush interval and has unwritten messages.
   */
  private def flushDirtyLogs(): Unit = {
    for ((topicPartition, log) <- currentLogs.toList ++ futureLogs.toList) {
        val timeSinceLastFlush = time.milliseconds - log.lastFlushTime
        // 当某个 log 自上次 flush 超过一定间隔后，会再次触发 flush
        if (timeSinceLastFlush >= log.config.flushMs)
          log.flush(false)
      } 
  }

  // flush （recoveryPoint, logEndOffset) 之间的所有日志。
  def flush(forceFlushActiveSegment: Boolean): Unit = flush(logEndOffset, forceFlushActiveSegment)
  
    /**
   * Flush local log segments for all offsets up to offset-1 if includingOffset=false; up to offset
   * if includingOffset=true. The recovery point is set to offset.
   *
   * @param offset The offset to flush up to; the new recovery point
   * @param includingOffset Whether the flush includes the provided offset.
   */
  private def flush(offset: Long, includingOffset: Boolean): Unit = {
    val flushOffset = if (includingOffset) offset + 1  else offset
    val newRecoveryPoint = offset

    // localLog.recoveryPoint 表示上次 flush 的 point，本次只需要 flush recoveryPoint 到 offset 之间的日志
      if (flushOffset > localLog.recoveryPoint) {
        localLog.flush(flushOffset)
        lock synchronized {
          // 更新 recoveryPoint
          localLog.markFlushed(newRecoveryPoint)
        }
      }
    }

/**
    * Flush this log segment to disk.
    *
    * This method is thread-safe.
    */
public void flush() throws IOException {
    LOG_FLUSH_TIMER.time(new Callable<Void>() {
        // flush 所有的 log segment 到磁盘
        public Void call() throws IOException {
            // 特别注意这里的 flush 顺序，先 flush log，再 flush index 文件，因为 index 文件可以基于 log 文件恢复。
            log.flush();
            offsetIndex().flush();
            timeIndex().flush();
            txnIndex.flush();
            return null;
        }
    });
}

// 当 flush 完成，会更新 recoveryPoint，但是这里并没有持久化 recoveryPoint
private[log] def updateRecoveryPoint(newRecoveryPoint: Long): Unit = {
    recoveryPoint = newRecoveryPoint
}
```

可以看到 flush 中并没有持久化 recoveryPoint，我们知道 recoveryPoint 之前的日志都已经 flush 了，也即到达了一个一致/正确的状态，如果 broker 宕机了，重启的时候只需要从 recoveryPoint 恢复即可。<br>
但是这里并没有持久化 recoveryPoint。<br>
实际上，还会有一个持久化 recoveryPoint 的后台任务，该任务就专门负责持久化所有 partition log 的 recoveryPoint，将其存放到一个文件当中，我们看看 kafka 对于该任务的注释：<br>

```text
  /**
   * Write out the current recovery point for all logs to a text file in the log directory
   * to avoid recovering the whole log on startup.
   */
```
> kafka 的一个优点就是注释很清晰，值得我们学习

recoveryPoint 文件的格式如下，实际上其他的 offsetCheckpoint 文件都以同样的格式保存：<br>

```text
/**
     * This class persists a map of (Partition => Offsets) to a file (for a certain replica)
     * The format in the offset checkpoint file is like this:
     * <pre>
     *  -----checkpoint file begin------
     *  0                <- OffsetCheckpointFile.currentVersion
     *  2                <- following entries size
     *  tp1  par1  1     <- the format is: TOPIC  PARTITION  OFFSET
     *  tp1  par2  2
     *  -----checkpoint file end----------
     *  </pre>
     */
```

这样当 broker 异常重启的时候，就可以直接从保存的 recoveryPoint 开始恢复日志了，这也是 recoveryPoint 命名的由来，用于加快日志的恢复。

# 日志恢复

我们看看日志是如何恢复的。

我们从 recoveryPoint 对应的 segment 开始，恢复每一个 segment，这里注意一个大原则：所有的其他信息都是基于 log 文件恢复的，log 文件是根本，是“一生二、二生三、三生万物”中的一。<br>
首先会清空 segment 对应的三种 index 文件，然后遍历 segment 中的每一个 batch，校验 batch 正确性以及构建 index 文件对应的 entry，当遍历完所有 logSegment 或者遇到一个不正确的 batch 后，<br>
恢复完毕。我们看看具体的代码实现：

```java
 /**
     * Run recovery on the given segment. This will rebuild the index from the log file and lop off any invalid bytes
     * from the end of the log and index.
     *
     *
     * @param producerStateManager Producer state corresponding to the segment's base offset. This is needed to recover
     *                             the transaction index.
     * @param leaderEpochCache Optionally a cache for updating the leader epoch during recovery.
     * @return The number of bytes truncated from the log
     */
    public int recover(ProducerStateManager producerStateManager, Optional<LeaderEpochFileCache> leaderEpochCache) throws IOException {
        // 清空所有的 index 文件
        offsetIndex().reset();
        timeIndex().reset();
        txnIndex.reset();

        int validBytes = 0;
        int lastIndexEntry = 0;
        maxTimestampAndOffsetSoFar = TimestampOffset.UNKNOWN;
        try {
            for (RecordBatch batch : log.batches()) {
                // 如果 batch 保存的 sum 以及重新计算得到的 sum 不相等，则抛出异常
                batch.ensureValid();
                ensureOffsetInRange(batch.lastOffset());

                // The max timestamp is exposed at the batch level, so no need to iterate the records
                if (batch.maxTimestamp() > maxTimestampSoFar()) {
                    maxTimestampAndOffsetSoFar = new TimestampOffset(batch.maxTimestamp(), batch.lastOffset());
                }

                // Build offset index
                // offsetIndex 和 timeIndex 都是稀疏索引
                if (validBytes - lastIndexEntry > indexIntervalBytes) {
                    offsetIndex().append(batch.lastOffset(), validBytes);
                    timeIndex().maybeAppend(maxTimestampSoFar(), shallowOffsetOfMaxTimestampSoFar());
                    lastIndexEntry = validBytes;
                }
                validBytes += batch.sizeInBytes();

                if (batch.magic() >= RecordBatch.MAGIC_VALUE_V2) {
                    leaderEpochCache.ifPresent(cache -> {
                        // 构建 leaderEpoch 信息，如果当前 batch 的 leaderEpoch > leaderEpoch.latestEpoch，则在 leaderEpoch 文件中加入一项：<batch.LeaderEpoch, batch.baseOffset>
                        if (batch.partitionLeaderEpoch() >= 0 &&
                                (!cache.latestEpoch().isPresent() || batch.partitionLeaderEpoch() > cache.latestEpoch().getAsInt()))
                            cache.assign(batch.partitionLeaderEpoch(), batch.baseOffset());
                    });
                    // 这里会更新 txnIndex 文件，实际上重建 txnIndex/producerStateManager 是需要遍历所有的 segments 的，
                    // 但是 kafka 通过 snapshot 的方式保存 producerStateManager 信息，从而加快 producerStateManager 的恢复。
                    updateProducerState(producerStateManager, batch);
                }
            }
        }

        return truncated;
    }
```

# 日志同步的流程

## 一些关键概念

- HW

HW(HighWatermark)，类似于 etcd 中的 commited index，表示 ISR(InSync Replicas) 中所有副本都同步了该 index 之前的消息。通常，consumer 仅仅可以消费 HW 之前的消息。<br>
之所以说是通常，因为在事务消息以及 consumer 隔离级别为 read_committed 场景下，consumer 仅可读取 LSO(Last Stable Offset) 之前的消息。

- LEO

Log End Offset

- LeaderEpoch

leaderEpoch 就是用来检测日志冲突的。partiton follower 每次 fetch 的时候，都会带上当前的 leaderEpoch。leader 节点接收到 fetch request 时，<br>
总是会检查自己日志维护的日志中，reqLeaderEpoch 对应的 endOffset 是不是小于 req.fetchOffset，如果是，则直接把这个 endOffset 发送给 follower。<br>
follower 收到请求后，就会截断自己的日志到 endOffset，然后重新发起 fetch 请求。

我们看看 leaderEpoch 相关逻辑：

```scala
  // 获取 leader 节点在客户端请求中的 epoch 的 endOffset 信息
  def lastOffsetForLeaderEpoch(currentLeaderEpoch: Optional[Integer], // 忽略
                               leaderEpoch: Int, // 客户端发送的 fetchRequest 中的 leaderEpoch
                               fetchOnlyFromLeader: Boolean): EpochEndOffset = {
      val localLogOrError = getLocalLog(currentLeaderEpoch, fetchOnlyFromLeader)
      localLogOrError match {
        case Left(localLog) =>
          // 直接在当前节点的 leaderEpochCache 中获取 leaderEpoch 对应的 endOffset
          localLog.endOffsetForEpoch(leaderEpoch) match {
            // 如果存在，设置 leaderEpoch 和 endOffset
            case Some(epochAndOffset) => new EpochEndOffset()
              .setPartition(partitionId)
              .setErrorCode(Errors.NONE.code)
              .setLeaderEpoch(epochAndOffset.leaderEpoch)
              .setEndOffset(epochAndOffset.offset)
            // 说明 leaderEpoch 还未结束，没有 endOffset
            case None => new EpochEndOffset()
              .setPartition(partitionId)
              .setErrorCode(Errors.NONE.code)
          }
        case Right(error) => new EpochEndOffset()
          .setPartition(partitionId)
          .setErrorCode(error.code)
      }
    }

  // 读取记录
  private def readRecords(
    localLog: UnifiedLog,
    lastFetchedEpoch: Optional[Integer],
    fetchOffset: Long,
    currentLeaderEpoch: Optional[Integer], // 忽略
    maxBytes: Int,
    fetchIsolation: FetchIsolation,
    minOneMessage: Boolean
  ): LogReadInfo = {

    lastFetchedEpoch.ifPresent { fetchEpoch =>
      // 获取当前节点维护的在 fetchEpoch 对应的 endOffset 信息
      val epochEndOffset = lastOffsetForLeaderEpoch(currentLeaderEpoch, fetchEpoch, fetchOnlyFromLeader = false)
      
      // 每次 fetch 都会进行日志冲突检测
      if (epochEndOffset.endOffset < fetchOffset) {
        // 直接将 endOffset 发送给 follower，follower 据此截断日志。
        val divergingEpoch = new FetchResponseData.EpochEndOffset()
          .setEpoch(epochEndOffset.leaderEpoch)
          .setEndOffset(epochEndOffset.endOffset)

        return new LogReadInfo(
          FetchDataInfo.empty(fetchOffset),
          Optional.of(divergingEpoch),
          initialHighWatermark,
          initialLogStartOffset,
          initialLogEndOffset,
          initialLastStableOffset)
      }
    }
  }

```

kafka 最开始基于 HW 判断是否阶段日志，这可能导致日志的丢失以及日志的不一致。通过引入 leaderEpoch 解决此问题。

# 如何发送 log

我们知道 kafka 通过 sendfile 发送日志，日志直接从文件发送给对应的套接字，无须拷贝到用户态内存以及避免了上下文切换，从而提高了性能。<br>
整个 readLog 逻辑就是用来确定从哪个 logSegment 文件的什么地方开始读以及读多少数据，最后通过 sendfile 把数据发送出去。