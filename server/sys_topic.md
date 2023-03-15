介绍 kafka 内部的两个重要的主题.

# __consumer_offsets
该主题有什么作用呢? 被 kafka 用来持久化消费者组的元信息和组消费位移信息. 我们都知道每一个消费者组都有一个对应的组协调器, 组协调器负责管理组的元信息和消费位移等信息.

在组协调器内部, 有个 offset cache 结构, 当消费者提交位移信息的时候, 会更新此 offset cache, 同时将消费者组的偏移信息作为一条新的消息持久化到 __consumer_offsets 中. 这样即使当前组协调器宕机, 当选举出新的组协调器的时候, 也可以根据 __consumer_offsets 中的信息生成正确的 offset cache. 注意, 往 __consumer_offsets 中写入消息的时候, acks = -1, 即当分区所有的副本都接受到此消息后,才认为消息写入成功.

offset cache 可以简单认为是一个 kv 结构, k 就是 groupid + topic + partition.

所以 __consumer_offsets 有什么用? 
- 持久化消费组的元信息和位移信息
- 利用 kafka 分区的副本机制, 提高可用性

# __transaction_state