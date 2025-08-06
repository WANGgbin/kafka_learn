描述 kafka 中的 checkpoint 机制。

checkpoint 机制其实跟快照的思想是类似的，将系统某个时刻的信息保存下来。这样系统异常退出的时候，就可以从上次的 checkpoint 开始恢复了，从而加速恢复流程。

在 kafka 中有很多的 checkpoint，包括：
- recoveryPoint
- leaderEpoch
- logStartOffset

关于 recoveryPoint，我们已经在 `log.md` 一文中进行了描述，本文我们看看其他的 checkpoint。

# checkpoint

## leaderEpoch

kafka 在 partition 中引入 leaderEpoch 的目的就是用于日志冲突监测，从而避免日志的丢失或者不一致。我们知道 leaderEpoch 保存的是该 partition 每个 leaderEpoch 与 baseOffset 的关联关系。<br>
如果不进行 checkpoint，在系统宕机重启的时候，就需要重头遍历 log 文件来重建内存中的 leaderEpoch 对象，性能会很差。

因此，kafka 通过 checkpoint 机制将 leaderEpoch 保存到一个文件中，文件内容如下：

```text
/**
 * This class persists a map of (LeaderEpoch => Offsets) to a file (for a certain replica)
 * <p>
 * The format in the LeaderEpoch checkpoint file is like this:
 * -----checkpoint file begin------
 * 0                <- LeaderEpochCheckpointFile.currentVersion
 * 2                <- following entries size
 * 0  1     <- the format is: leader_epoch(int32) start_offset(int64)
 * 1  2
 * -----checkpoint file end----------
 */
```

不过需要注意的是，leaderEpoch 的 Checkpoint **并不是通过定时的方式更新的**，而是在每次 append log 的时候，会判断是否需要新添加一项，如果需要就直接更新 leaderEpochCache 并 flush 到对应的 checkpoint 文件中。<br>
我们看看代码实现：

```java
public void assign(int epoch, long startOffset) {
    EpochEntry entry = new EpochEntry(epoch, startOffset);
    // 添加到 cache 并 flush 到文件中。
    if (assign(entry)) {
        writeToFile();
    }
}

private boolean assign(EpochEntry entry) {

    // 判断是否需要添加到 cache
    if (!isUpdateNeeded(entry)) return false;

    lock.writeLock().lock();
        // 加锁后，现在不一定需要添加，因此还需要判断依次
    if (isUpdateNeeded(entry)) {
        epochs.put(entry.epoch, entry);
        return true;
    } else {
        return false;
    }
    lock.writeLock().unlock();
}

// 我们看看 isUpdatedNeeded 的实现
private boolean isUpdateNeeded(EpochEntry entry) {
    return latestEntry().map(epochEntry -> entry.epoch != epochEntry.epoch || entry.startOffset < epochEntry.startOffset).orElse(true);
}
```

## 其他 checkpoint

其他的 checkpoint 后面有空可以再看。

# 文件替换流程

在 checkpoint 的时候，可能会直接使用新的数据覆盖原来的文件，比如 recoveryPoint 的 checkpoint。这里的一个问题是，如果直接在原来的文件写，在写到一半的时候如果出错，就会导致之前的数据丢失了。<br>
kafka 采用的思路是：

- 先创建一个 tmp 文件，写 tmp 文件
- 然后通过操作系统的 mv 能力，原子性的使用 tmp 文件替换 checkpoint 文件。已经打开就文件的进程仍然可以正常访问旧文件。
当所有的进程都关闭文件后，旧的文件才会真正删除。

如此一来，即使写 tmp 文件失败，也没关系，原来 checkpoint 文件中的数据不受影响，后面可以再次 checkpoint 重试。