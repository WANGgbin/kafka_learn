描述 kafka 中 producer 相关特性。

# 架构

kafka producer 整体包括两部分：producer 主线程、sender 线程。<br>

主线程负责把数据发送到 RecordAccumelator，sender 线程从 RecordAccumelator 读取数据并发送给 broker.

## producer 线程

步骤如下：

- 获取集群元信息

    只有知道了集群元信息，才知道 topic + partition 应该发送给哪个 broker，至于如何维护集群元信息，后面集群元信息部分会介绍。


- 拦截器

    对发送的消息进行进一步处理。

- key、value 序列化

- 分区器

    根据一定的分区策略，将消息发送到某个 partition 对应的 RecordAccumelator。


## sender 线程

步骤如下：

- 首先判断哪些 broker 还可以接受数据

    每个到 broker 的连接有一个配置：max.inflight.requests.per.connection，表示一个连接同时已发送但未接收到的最大请求数量。这个数量未达到上限的
    时候才可以继续给 broker 发送消息。

- 从 recordAccumelator 读取数据，构建 Requests 加入到 inflightRequests 队列中，同时发送给 broker。

    这里需要注意一点：因为一个 broker 可能同时保存了多个 partition leader 副本，因此，为了尽可能均匀的从若干个 partition recordAccumelator 读取数据，
    维护了一个 index，表示下次开始读取请求的 partition 的 index，而不是每次都从第一个 partition accumelator 读取数据。<br>

    此外，我们从 accumlator 读取了数据，但是发送失败了怎么办？这个时候 accumlator 中已经没有之前的数据了，怎么重新发送呢？引入了 inflightBatch 的方法解决
    此问题。TODO: 待确认。

- 当接收到来自 broker 的 resp 的时候，删除 inflightRequests、inflightBatch 中对应的对象，同时调用用户注册的 callback 函数，完成一次请求的发送。


# 为了保证高性能，都做了什么

- 批量发送

    如果一个一个 key/value 的发，带宽的利用率太低了。为了提高带宽利用率，通过批量的方式发送消息。

- 压缩

    为了避免消息体过大导致网络 i/o 压力过大，在发送请求前还会压缩数据。

- 一条 tcp 连接可以同时发送多个请求

    如果一个 tcp 连接同时只能存在一个 inflightRequests，那整个吞吐是比较差的。就跟未开启 pipeline 的 http 请求一样。

# 集群元信息更新

正确的集群元信息是 producer 能够将请求发送给正确 broker 的前提。因此在每次发送前，我们都应该确保当前缓存的集群元信息是有效的。<br>

既然是缓存，我们需要考虑以下几个问题：

- 什么时候会刷新 cache

    producer 通过定时更新的方式更新 元信息 cache。定时是兜底的方式，实际上在下次更新 cache 前，很可能发生一些 超时、连接断开等错误。这种场景下，也会更新 cache,
    因为目标节点可能宕机，因此需要获取新的 leader broker。<br>

    除此之外，当我们的请求发送给错误的 nodeID 的时候(partition 对应的 leader 发生了改变)，broker 会告诉我们 partition 目前的 leader broker，producer 会据此
    更新 cache 信息。

- 如何刷新 cache

    sender 线程每次在 poll 之前，都会判断是否发送 metaDataUpdateRequest。判断标准是：

    - 到期
    - 发生了一些需要更新 metaData 的事件，这些事件会设置一个标记字段，sender 线程通过检查该字段决定是否更新 metaData
    - 如果已经有一个 inflight 的更新请求，则本次不再发送更新请求

- 请求发送给 kafka 中哪个节点呢

    发送给负载最小的节点。负责最小的节点判断标注是什么呢？<br>

    我们知道每个 broker 都对应一个 inflightRequests Dequeue，inflightRequests 中请求数量最小的 broker，我们认为就是负载最小的节点。


# 如何实现 exactly once 语义

这里的有序指的是 partition 维度的有序。broker 会给 producer 分配一个 producerID，同时 producer 会为每个分区维护一个 batch sequence number，每一个 batch 对应一个 sequence number,
同时 broker 端会为每个 producer + partition 保存上一次 batch 的 sequence number，每当新写入一个 batch 的时候，只有 batch 的 new_seq == old_seq + 1，broker 才会保存该消息，从而保证了消息的 exactily once 语义。

如果要打开幂等生产的特性，需要设置 `enable.idempotence=true`，不过从 kafka 3.0 开始这个选项是默认打开的。<br>


# 如何重试

通过设置 `retries` 参数，当请求发送失败的时候 producer 内部即可进行重试，无须用户手动重试。那么重试的大概逻辑是什么呢？<br>

在发送请求失败的时候，最后都会构造一个 response(即使未从 broker 接收到 response，也会根据错误类型比如超时构造一个 response，用于后续的同一处理)，最后会调用 completeBatch 完成重试逻辑，代码如下：

```java
 /**
     * Complete or retry the given batch of records.
     *
     */
    private void completeBatch(ProducerBatch batch, ProduceResponse.PartitionResponse response, long correlationId,
                               long now, Map<TopicPartition, Metadata.LeaderIdAndEpoch> partitionsWithUpdatedLeaderInfo) {
        Errors error = response.error;

        if (error != Errors.NONE) {
            // 判断能否重试，条件包括：是否是可重试错误、重试次数是否达到上限等。
            if (canRetry(batch, response, now)) {
                // 从 inflightBatch 中删除 batch，并将 batch 重新加入到 partition dequeu head，从而后续可以走正常的发送流程并首先发送该 batch
                reenqueueBatch(batch, now);
            } 
        }

        // 重试可能导致的问题就是消息的乱序，为了解决乱序的问题，我们可以将 max.inflight.requests.per.connection 设置为 1，即这里的 guaranteeMessageOrder == true
        // 这样任何时候一条连接只能有一个 inflightRequest，即使重试，也不会导致乱序问题。
        // 在 guaranteeMessageOrder == true 的时候，发送一个请求的时候，就会将 partition 标记为 mute，只有接收到请求的响应后，才会标记为 unmute，这样后续就可以继续从该 partition 读取消息了。
        if (guaranteeMessageOrder)
            this.accumulator.unmutePartition(batch.topicPartition);
    }
```

# 如何保证消息有序

将 max.inflight.requests.per.connection 设置为 1 即可。

# 请求发送超时如何实现的

因为 sender 线程是一个 dameon thread，会在一个死循环中一直执行 sendRequest、poll 逻辑。因此，请求超时并没有通过 kafka 的 timewheel 实现，而是通过在每一轮循环中主动检查的方式获取过期的 requests。
我们来看看代码：
```java
public List<ClientResponse> poll(long timeout, long now) {

        // process completed actions
        long updatedNow = this.time.milliseconds();
        List<ClientResponse> responses = new ArrayList<>();
        // 这一步会判断 inflightRequests 中超时的 requests，关闭对应的 tcp 连接，并构建超时的 resp 添加到 responses 中。
        handleTimedOutRequests(responses, updatedNow);
        // 这一步就会根据 response 处理超时的 requests
        completeResponses(responses);

        return responses;
    }


    /**
     * Iterate over all the inflight requests and expire any requests that have exceeded the configured requestTimeout.
     * The connection to the node associated with the request will be terminated and will be treated as a disconnection.
     * 
     * kafak 的注释说的很清楚了。
     */
    private void handleTimedOutRequests(List<ClientResponse> responses, long now) {
        List<String> nodeIds = this.inFlightRequests.nodesWithTimedOutRequests(now);
        for (String nodeId : nodeIds) {
            // close connection to the node
            this.selector.close(nodeId);
            // 这一步实际上是构造超时的 resp，并添加到 response 中。
            // 这个思想很值得我们学习，虽然我们并没有真正收到来自 broker 的 resp，但通过模拟一个 resp，从而可以在后续处理 resp 的流程中
            // 同一处理超时的错误。
            // 优秀程序员的一大能力就是：抽象。
            processTimeoutDisconnection(responses, nodeId, now);
        }
    }
```

# 连接管理

producer 什么时候建立到目标节点的 tcp 连接呢？<br>

sender 线程每次 sendRequest 之前，会从 accumlator 读取数据从而决定需要往哪些 node 发送请求。这个时候会判断 isReady(node)，如果为 false，
同时判断能否发起连接(如果尝试了很多次，就会有 backoff，避免频繁发起连接)，如果可以就会初始化到目标节点的连接，当然这个过程是异步的，不阻塞 sender 主流程。 

