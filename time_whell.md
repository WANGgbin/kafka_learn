描述 kafka 中的时间轮算法。

时间轮算法常用于 延时/定时任务。凡是延时/定时任务实现，就涉及下面几个问题：

- 插入/删除 任务的时间复杂度
- 时间精度
- 时效性
- 时间范围

# 算法描述

时间轮的思想类似于日常生活中的表盘。时间轮有多个槽，槽的时间跨度决定了时间轮的精度，我们记为 ticketMs。比如槽的大小为 5ms，那么此时间轮
能处理的延时任务的时间精度就是 5ms。此外，时间轮还有个指针，随着该指针的转动，处理到期的任务。假设指针表示的时间为：currentMs，
那么整个时间轮能表示的时间跨度就是 currentMs + (slotNum * ticketMs)。

每个槽对应一个 taskList，表示到期时间在 [slotMs, slotMs + ticketMs) 的所有任务。


- 插入/删除 时间复杂度

当我们要插入一个任务的时候，只需要根据到期时间计算任务所在的 slot，然后加入到对应的 taskList 中即可，因此时间复杂度是 O(1)。我们知道还可以
使用 heap 的方式实现定时任务，go runtime 就是通过 heap 实现 timer 的。但这种算法的时间复杂度为 O(logN)，性能较差。

- 时间精度

由槽大小决定。


- 时效性

时间轮的指针还是通过定时任务的方式驱动的，定时任务的间隔，决定了触发到期任务的时效性。

- 时间范围

一层时间轮能表示的时间范围是有限的，为了支持跨度更大的时间范围，引入了 **多层时间轮** 的概念，如果不对层数限制的话，理论上可以支持无穷大的时间范围。


# 应用场景

kafka 很多场景中都可以看到延时任务的影子。举几个例子：

## DelayJoin

在 consumer group rebalance 的时候，consumer 会给 group coondinator 发送 JoinGroupRequest，group coondinator 在接收到 JoinGroupRequest 的时候
并不会立即发送 Resp 给 consumer，因为 leader consumer 的 JoinGroupResp 中需要包含 consumer group 中的所有成员信息，这样 leader consumer 才可以分配分区，
因此，只有当所有的 consumer 都发送 JoinGroupRequest 后或者超时后(reblance.timeout.ms) 才发送 JoinGroupResponse 给 consumer。实际上在 consumer group
首次初始化的时候，只能等到超时才发送 Resp，因为此场景下，我们无法知道有多少 consumer 要加入到 consumer group。<br>

DelayJoin 的实现就是将一个 DelayJoin 对象扔到 timer 中，每当接收到 consumer 的 JoinGroupRequest 的时候，就会尝试判断下 DelayJoin 能否执行，因为当所有的
comsumer 都发送 Request 后，就可以执行 DelayJoin 即发送 JoinGroupResp 了，无须等到超时。


## DeplayProduce

当 producer 指定了 syncs 参数后，只有跟 syncs 数量匹配的副本都同步数据后，才可以给 producer 发送响应。所以也会创建一个延时任务。当接收到副本的 FetchRequest
的时候，就会尝试判断下 DelayProduce 能不能执行，即是否满足 syncs 数量的副本都同步了数据，如果是，则取消延时任务并执行 DelayProduce 即给 producer 发送 Resp。