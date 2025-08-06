# kafka 中消息到底会不会丢

会丢。我们从三个方面来分析。

- producer

    如果 acks 参数设置为 0 或者 1，都有可能导致消息丢失。比如消息压根没发送到 broker，或者消息只发送到 master broker 还没有同步到 slave broker，但是随后 master broker dump 了，选举的新的 master broker<br>
    就是丢失数据的。

    那么如果把 acks 参数这是为 -1，是不是就不会丢失数据了。也不一定。如果 ISR 中只有一个副本的时候，此时等价于 acks = 1。

- broker

    如果允许不在 ISR 之中的副本也可以成为 leader，也会导致数据丢失。

- consumer

    如果打开 enable.auto.commit，一个消息可能消费失败了，但是已经自动提交了，后续就消费不到此数据，进而数据丢失。

# kafka 为了实现高吞吐，都做了什么

我们也从几个方面来分析。

- producer

    - batch 特性

        并不是有一条数据就发送，而是攒够一个 batch 后再发送，可以提高带宽利用率，进而提高吞吐。

    - compress 特性

        为了进一步提高带宽利用率，还会对数据进行压缩。

    - 一条连接可以同时发送多个 request

        在 http 中，一般一个连接同时只能有一个 inflight 的请求，发送请求接受响应，然后再发送下一个请求。但这样无法充分利用两端的处理能力。<br>
        kafka 中，一个 tcp 连接可以同时有多个 inflight 的请求，充分利用双端的能力，进一步提高系统吞吐。

- broker

    - sendfile

        无论是 slave 从 master fetch 数据，还是 consumer master fetch 数据。kafka 都是通过 sendfile 的方式直接将数据从文件发送到套接字，<br>
        无须用户态数据拷贝以及用户态/内核态切换，大大提高系统性能。

    - partition

        我们知道一个 topic 下可以有多个 partition。多个 partition 可以充分利用多机的能力，进一步提高系统的写吞吐。

    - 磁盘顺序写入

        kafka 通过磁盘顺序写入的方式大大提高了磁盘写入的性能。

- consumer

    - 批量拉取

        一次 fetch 可以拉取多个消息

    - 并行消费

        一个 consumer group 中的多个 consumer 可以同时消费不同的 partition
        

# 消息堆积了，通常是什么原因？如何排查？

- 频繁重平衡

    我们知道在重平衡的过程中，consumer group 是无法消费消息的，如果 consumer group 频繁的进行重平衡，就会导致数据的堆积。<br>
    通常在 topic/分区数量发生变化、consumer 上下线的时候都会导致重平衡。但实际场景中，由 consumer 异常退出、消费消息时间过长导致的 rebalance 占多数。

- 写入 QPS 变高

- 消费变慢

    有时候即使没有变动消费者逻辑，也可能导致消费变慢。比如 db 中数据达到一定阈值，没有创建索引导致满查询。又或者消费者的下游有变动等。