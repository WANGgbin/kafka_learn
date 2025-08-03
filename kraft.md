描述 kafka 中 raft 的使用。

# 为什么引入 kraft 不再使用 zookeeper

使用 zookeeper 有以下几个问题：

- 部署/运维复杂，需要单独部署 zookeeper 组件
- kafka 强耦合 zookeeper，zookeeper 容易成为 kafka 的短板，但却无法修改 zookeeper

因此通过 raft 协议，来维护集群的元信息，不再使用 zookeeper.

# kraft 元信息同步流程

在 kraft mode 模式下，集群中的节点可以扮演两种角色：一种是 broker、另一种是 controller。所有的 controller 节点组成 kraft 集群。<br>
在 kafka 集群中，集群元信息是很重要的，集群元信息决定了一个 broker 应该保存哪些副本，应该从哪个 leader 节点 fetch 消息等等。<br>
关于集群元信息我们主要关注两个方面：如何保存、如何传播。

kraft 模式下，kafka 通过 __cluster_meta topic 来保存集群元信息，该 topic 只有一个 partition。与普通 topic 不同的是，当 kraft 集群中的<br>
多数节点都同步了 topic 信息后，leader controller 就会更新自己的 hw。这样普通的 broker 节点就可以 fetch 到该 topic 信息，然后 apply 到内存中的集群元信息，从而实现元信息的传播。<b>

# kraft __cluster_meta 清理

__cluster_meta 中的每个 record 保存的是集群元信息的增量内容而不是全部内容。随着集群的运行，该 topic 记录越来越多，应该清理哪些记录呢？

controller 节点会运行一个定时任务，该任务会基于 topic 当前的 commited offset 生成一个 snapshot 文件，该文件保存了 commited offset 之前所有 record 对应的集群元信息。<br>
当 snapshot 文件生成后，commited offset 之前的 topic record 就可以清除了。

实际上这里的 __cluster_meta 与 snapshot 的关系与 etcd 中 WAL 与 bbolt 的关系类似，都可以看作是应用层的存储。__cluster_meta 与 WAL 都是 raft 中用来同步的 log，<br>
log 同步后需要 apply 到应用层。

# 一个新的节点如何知道集群的 leader 节点

当我们在集群中加入一个新的节点的时候，这个节点需要跟 leader 节点通信同步 __cluster_meta 日志。那么新节点如何发现 leader 节点呢？

在启动一个节点的时候，通常会配置 `controller.quorum.bootstrap.servers` 信息，该信息就是集群中参与 raft 选举的节点信息。当节点启动的时候，<br>
会随机给 bootstrap.srevers 中的一个节点发送 `FetchRequest` 消息，该节点会把自己知道的 leader 信息以 `FetchResponse` 的形式发送给新节点，<br>
后续新节点就可以跟 leader 正常同步信息了。

# broker 节点注册流程是什么样的

新节点运行的时候，先找到 leader 节点，然后给 leader 节点发送注册节点请求，这个信息会更新到 __cluster_meta 中，待 raft 集群中多数节点同步后，就可以应用到整个 kafka 集群中了。

# controller 如何监测 broker 下线

broker 会定时给 leader 发送 heartbeat 消息，当超时后，leader 任务 broker 下线，将信息更新到 __cluster_meta 中，然后应用到整个 kafka 集群中。

# 分布式场景下的一些思考

## 集群元信息

无论什么类型的集群，都会有集群元信息，通常包括集群节点信息、节点的主从信息等等。<br>
集群元信息通常是需要持久化的，这里的持久化不是单机的磁盘持久化，这在单机故障的时候，还是存在不可用的问题。<br>
通常会在多个节点都进行存储，从而避免单点故障问题。

比如：k8s 集群元信息就存在在 etcd 中，etcd 就是个分布式的强一致的 k/v 数据库。<br>
又比如，kafka raft 模式中，集群元信息就存储在每个节点的 log 文件中。

## 集群元信息同步

通常集群的每个节点或多或少都需要关注集群元信息，这就涉及集群元信息的同步。<br>
通常有一个组件维护集群元信息，这个组件要能容错，比如 raft 算法实现的集群。<br>
其他节点通过不断 poll(轮询) 的方式这个组件 fetch 集群元信息。

有了集群元信息同步，当集群元信息发生变化，所有节点都能感知到(比如 producer/consumer)，从而保证了集群行为的正确性(发送消息到正确的 broker，从正确的 broker 拉取消息)。

## 集群节点变更(上/下线)

节点变更属于集群元信息变更的一部分。

通常节点下线监控的实现思路是：每个节点不断给 leader 节点发送心跳，当心跳超时时，便认为节点下线，然后更新并同步集群元信息。<br>
节点注册的思路是：预先指定(配置文件/命令行)集群中的一些节点(可以是 leader 可以不是 leader)，通常集群中的节点都知道 leader 是谁(因为有元信息同步)，所以<br>
只要给集群中任一节点发送请求，通常都能获取到 leader 节点。