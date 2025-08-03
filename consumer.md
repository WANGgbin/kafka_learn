描述 kafka 中消费端相关逻辑。

# __consumer_state

kafka 专门使用一个内部主题 __consumer_state 来保存消费组的信息。__consumer_state 的每个分区所在的 broker 也称为 consumer group coordinator。<br>
那么如何确定一个消费组对应的 coordinator 是哪个呢？<br>
每个消费组对应一个 groupID，计算方式为 hash(groupID) % partitionCnt。

# 重平衡

当一个消费组内的成员数量发生变化或者消费的 topic 的分区数量发生变化的时候，就需要进行重平衡。重平衡的目的是进行分区的重新分配分区与消费者的对应关系，尽量实现负载均衡。

## 重平衡流程

这里以加入一个新节点来描述重平衡的流程。

### FindCoordinator

新加入的节点不知道消费组的 coordinator 是谁，所以第一步需要查找对应的 coordinator。给负载最小的节点 broker (TODO：待确定)发送 FindCoordinator 请求，broker 收到请求后,<br>
通过前面介绍的分配算法计算到消费组对应的 coordinator，然后把 coordinator 信息发送给消费者。

### JoinGroup

当消费者确定了 coordinator 后，就会给 coordinator 发送 JoinGroupRequest 表明自己想加入到 group 当中。coordinator 接收到 JoinGroupRequest 后，当消费组其他节点发来 heartBeatRequest<br>
的时候，coordinator 就会在 heartBeatResponse 加入信息告诉消费者，当前消费组要进行重平衡了。消费者收到后，就会给 coordinator 发送 JoinGroupRequest。

coordinator 接收到 JoinGroupRequest 后，实际上并不会立即给消费者发送 JoinGroupResponse。为什么呢？<br>
我们知道 coordinator 会给 消费组的 leader 节点的 JoinGroupResponse 中发送消费组成员信息，从而执行分区在消费组中的分配逻辑。<br>
因此只有当消费组中所有成员都加入或者超时的时候，才会给 leader 发送 JoinGroupResponse 信息。可是对于其他非 leader 节点，为什么也是延迟发送 Response 呢？<br>
想象一下在新一轮的 rebalance 中，如果给非 leader 节点立即回复 Response，但是 leader 节点下线了，那岂不是无法进行分区分配了？<br>
因此，所有节点的 JoinGroupResponse 都是延迟发送的，当超时后，原来的 leader 节点还没有发送 JoinGroupRequest，则会在已经 Join 的节点中选择一个新的 leader。<br>
延时任务的时间对应的参数就是 group.rebalance.timeout.ms(消费组所有成员最大的 rebalance.timeout.ms)。

我们大概看看 coordinator 处理 JoinGroupRequest 的代码：

```java
private CoordinatorResult<Void, CoordinatorRecord> maybePrepareRebalanceOrCompleteJoin(
        ClassicGroup group,
        String reason
    ) {
        // group 为 stable、completingRebalance、empty，才可以进行 rebalance
        if (group.canRebalance()) {
            return prepareRebalance(group, reason);
        } else {
            // 已经在 prepareRebalance 状态下，接收到 joinGroupRequest，会尝试结束 join phase
            return maybeCompleteJoinPhase(group);
        }
    }

 // 进入 rebalance 状态
 CoordinatorResult<Void, CoordinatorRecord> prepareRebalance(
        ClassicGroup group,
        String reason
    ) {

        boolean isInitialRebalance = group.isInState(EMPTY);
        if (isInitialRebalance) {
            // The group is new. Provide more time for the members to join.
            int delayMs = classicGroupInitialRebalanceDelayMs;
            int remainingMs = Math.max(group.rebalanceTimeoutMs() - classicGroupInitialRebalanceDelayMs, 0);

            // 对于 intialRebalance，会创建一个比 rebalanceTimeout 更小的延时任务，
            // 该延时任务到期后，会判断该时间段内是否有新的节点加入，如果有，则再创建一个同样的延时任务。
            // 当无新的节点加入或者 remaingMs == 0 的时候，结束 join phase

            // 为什么不直接创建个 delayMs == rebalanceTimeout 的延时任务呢？
            // 个人理解，如果没有新的节点加入了但是延时任务还未到期就只能等着，从而导致 join 流程更慢。
            // 因此，在判断没有新的节点加入的时候就直接结束 join phase.
            // 那怎么判断有没有新的节点加入呢？那就创建一个相对 rebalanceTimeout 较小的延时任务，该延时任务到期
            // 如果没有接收到新的节点，我们就认为不再有新的节点加入，结束 join phase 
            timer.schedule(
                classicGroupJoinKey(group.groupId()),
                delayMs,
                TimeUnit.MILLISECONDS,
                false,
                () -> tryCompleteInitialRebalanceElseSchedule(group.groupId(), delayMs, remainingMs)
            );
        }

        group.transitionTo(PREPARING_REBALANCE);

        log.info("Preparing to rebalance group {} in state {} with old generation {} (reason: {}).",
            group.groupId(), group.currentState(), group.generationId(), reason);

        // 对于其他类型的 rebalance，则直接判断所有节点是否都已经发送 join request，如果是则结束 join phase，否则刷新延时任务
        return isInitialRebalance ? EMPTY_RESULT : maybeCompleteJoinElseSchedule(group);
    }
```

我们接着看看 `tryCompleteInitialRebalanceElseSchedule` 的实现：

```java
private CoordinatorResult<Void, CoordinatorRecord> tryCompleteInitialRebalanceElseSchedule(
        String groupId,
        int delayMs,
        int remainingMs
    ) {
        ClassicGroup group;
        group = getOrMaybeCreateClassicGroup(groupId, false);

        // 如果期间有新的节点加入且 remaingMs 不为 0，则说明可能还会有新的节点加入，那就再创建个延时任务
        if (group.newMemberAdded() && remainingMs != 0) {
            // A new member was added. Extend the delay.
            group.setNewMemberAdded(false);
            int newDelayMs = Math.min(classicGroupInitialRebalanceDelayMs, remainingMs);
            int newRemainingMs = Math.max(remainingMs - delayMs, 0);

            timer.schedule(
                classicGroupJoinKey(group.groupId()),
                newDelayMs,
                TimeUnit.MILLISECONDS,
                false,
                () -> tryCompleteInitialRebalanceElseSchedule(group.groupId(), newDelayMs, newRemainingMs)
            );
        } else {
            // 否则结束 join phase，给所有的 member 发送 JoinGroupResponse，并将 consumer group 状态更改为 completingRebalance
            return completeClassicGroupJoin(group);
        }

        return EMPTY_RESULT;
    }
```

接着我们看看 `completeClassicGroupJoin` 的实现：

```java
private CoordinatorResult<Void, CoordinatorRecord> maybeCompleteJoinPhase(ClassicGroup group) {
        // 对于 initial rebalance，该函数啥都不会发生。
        // 对于其他 rebalance，如果当前所有节点都已经发送了 joinGroupRequest，则结束 join phase
        if (group.isInState(PREPARING_REBALANCE) &&
            group.hasAllMembersJoined() &&
            group.previousState() != EMPTY
        ) {
            return completeClassicGroupJoin(group);
        }

        return EMPTY_RESULT;
    }

public boolean hasAllMembersJoined() {
        // 当接收到一个 member 的 joinGroupRequest 的时候， numMembersAwaitingJoinResponse++
        // 这里表达的含义就是当 coordinator 接收到所有 member 的 joinGroupRequest 的时候，即所有的成员都已经加入，此时就可以结束 join phase
        return members.size() == numMembersAwaitingJoinResponse; 
    }

```

接着我们看看 `completeClassicGroupJoin` 的实现：

```java
private CoordinatorResult<Void, CoordinatorRecord> completeClassicGroupJoin(
        ClassicGroup group
    ) {
        // 这一步很重要，取消对应的延时任务
        timer.cancel(classicGroupJoinKey(group.groupId()));
        String groupId = group.groupId();

        // 删除所有尚未发送 joinGroupRequest 的 member 集合
        Map<String, ClassicGroupMember> notYetRejoinedDynamicMembers =
            group.notYetRejoinedMembers().entrySet().stream()
                .filter(entry -> !entry.getValue().isStaticMember())
                .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));

        if (!notYetRejoinedDynamicMembers.isEmpty()) {
            notYetRejoinedDynamicMembers.values().forEach(failedMember -> {
                group.remove(failedMember.memberId());
                timer.cancel(classicGroupHeartbeatKey(group.groupId(), failedMember.memberId()));
            });

            log.info("Group {} removed dynamic members who haven't joined: {}",
                groupId, notYetRejoinedDynamicMembers.keySet());
        }

        // 原来的 leader 可能没有发送 joinGroupRequest，则选举新的 leader
        if (!group.maybeElectNewJoinedLeader() && !group.allMembers().isEmpty()) {
            timer.schedule(
                classicGroupJoinKey(groupId),
                group.rebalanceTimeoutMs(),
                TimeUnit.MILLISECONDS,
                false,
                () -> completeClassicGroupJoin(group.groupId())
            );

            return EMPTY_RESULT;
        } else {
            // 这里就是生成一个新的 generationID
            group.initNextGeneration();
            // 给每个节点发送 JoinGroupResponse
            group.allMembers().forEach(member -> {
                List<JoinGroupResponseData.JoinGroupResponseMember> members = Collections.emptyList();
                // 这一点很重要，对于 leader，需要发送消费组的成员信息，从而能够进行分区分配
                if (group.isLeader(member.memberId())) {
                    members = group.currentClassicGroupMembers();
                }

                JoinGroupResponseData response = new JoinGroupResponseData()
                    .setMembers(members)
                    .setMemberId(member.memberId())
                    .setGenerationId(group.generationId())
                    .setProtocolName(group.protocolName().orElse(null))
                    .setProtocolType(group.protocolType().orElse(null))
                    .setLeader(group.leaderOrNull())
                    .setSkipAssignment(false)
                    .setErrorCode(Errors.NONE.code());

                // 给 member 发送 joinGroupResponse
                group.completeJoinFuture(member, response);
                rescheduleClassicGroupMemberHeartbeat(group, member);
                member.setIsNew(false);

                group.addPendingSyncMember(member.memberId());
            });

            // 创建 sync 阶段的延时任务，在延时时间到期后，如果还未接收到所有 member 的 syncRequest，则重新发起 rebalance
            // 具体逻辑，我们在 syncGroup 小节描述
            schedulePendingSync(group);
        }

    return EMPTY_RESULT;
}
```

最后我们看看非 initialRebalance 的 prepareRebalance 流程是啥样的：

```java
private CoordinatorResult<Void, CoordinatorRecord> maybeCompleteJoinElseSchedule( ClassicGroup group ) {
    String classicGroupJoinKey = classicGroupJoinKey(group.groupId());
        // 本质就是创建一个延时任务，延时时间等于 group 所有 member 中最大的 rebalanceTimeout
        timer.schedule(
            classicGroupJoinKey,
            group.rebalanceTimeoutMs(),
            TimeUnit.MILLISECONDS,
            false,
            () -> completeClassicGroupJoin(group.groupId())
        );
}

public int rebalanceTimeoutMs() {
    int maxRebalanceTimeoutMs = 0;
    for (ClassicGroupMember member : members.values()) {
        maxRebalanceTimeoutMs = Math.max(maxRebalanceTimeoutMs, member.rebalanceTimeoutMs());
    }
    return maxRebalanceTimeoutMs;
}

```

总体来说，joinGroup 又是一个 `延时任务`的典型场景，我们知道 kafka 中有很多的延时任务，包括延时生产、延时 join 等等。

### SyncGroup

当 coordination 给消费组所有成员发送完 JoinGroupResponse 后，consumer group 进入 `CompletingRebalance` 状态，当接收到 leader 的 SyncGroupRequest 后，consumer group <br>
进入 `Stable` 状态。


```java
private CoordinatorResult<Void, CoordinatorRecord> classicGroupSyncToClassicGroup(
        ClassicGroup group,
        RequestContext context,
        SyncGroupRequestData request,
        CompletableFuture<SyncGroupResponseData> responseFuture
    ) throws IllegalStateException {
        String groupId = request.groupId();
        String memberId = request.memberId();

        if (group.isInState(COMPLETING_REBALANCE)) {
            // 如果 group 还在 completingRebalance 状态，表示还没有收到 leader 的分区信息，暂时挂起，等接收到 leader 分区信息的时候，再
            // 发送 SyncGroupResponse
            group.member(memberId).setAwaitingSyncFuture(responseFuture);
            // 表示接受到了来自 member 的 SyncGroupRequest
            removePendingSyncMember(group, request.memberId());

            // 如果接收到来自 leader 的 SyncGroupRequest，获取并持久化分区信息，同时将消费组的状态切换为 stable，同时
            // 给阻塞的 member 发送 SyncGroupResponse
            if (group.isLeader(memberId)) {

                // Fill all members with corresponding member assignment. If the member assignment
                // does not exist, fill with an empty assignment.
                Map<String, byte[]> assignment = new HashMap<>();
                request.assignments().forEach(memberAssignment ->
                    assignment.put(memberAssignment.memberId(), memberAssignment.assignment())
                );

                // 对于没有分区的 member，分区信息置位空
                // 我们知道一个分区只能由一个消费者消费，如果消费者数量 > 分区数量，就有可能发生这样的清空
                Map<String, byte[]> membersWithMissingAssignment = new HashMap<>();
                group.allMembers().forEach(member -> {
                    if (!assignment.containsKey(member.memberId())) {
                        membersWithMissingAssignment.put(member.memberId(), EMPTY_ASSIGNMENT);
                    }
                });
                assignment.putAll(membersWithMissingAssignment);

                CompletableFuture<Void> appendFuture = new CompletableFuture<>();
                appendFuture.whenComplete((__, t) -> {
                    if (group.isInState(COMPLETING_REBALANCE) && request.generationId() == group.generationId()) {
                        // 给所有等待 SyncGroupResponse 的节点发送 SyncGroupResponse
                        setAndPropagateAssignment(group, assignment);
                        // 消费组状态切换为 Stable
                        group.transitionTo(STABLE);
                    }
                });

                return
            }
        } else if (group.isInState(STABLE)) {
            removePendingSyncMember(group, memberId);
            // 如果消费组状态未 stable，则直接发送 SyncGroupResponse
            ClassicGroupMember member = group.member(memberId);
            responseFuture.complete(new SyncGroupResponseData()
                .setProtocolType(group.protocolType().orElse(null))
                .setProtocolName(group.protocolName().orElse(null))
                .setAssignment(member.assignment())
                .setErrorCode(Errors.NONE.code()));
        } 

        return EMPTY_RESULT;
    }
```

这里就有个问题，如果一直接收不到某个 member 的 SyncGroupRequest 怎么办？显然是需要重新发起 Rebalance 的。而在前面介绍 CompleteJoin 的时候，最后会创建一个延时任务：

```java
// 延时任务时间为 group.rebalanceTimeout
private void schedulePendingSync(ClassicGroup group) {
        timer.schedule(
            classicGroupSyncKey(group.groupId()),
            group.rebalanceTimeoutMs(),
            TimeUnit.MILLISECONDS,
            false,
            () -> expirePendingSync(group.groupId(), group.generationId()));
    }

private CoordinatorResult<Void, CoordinatorRecord> expirePendingSync(
        String groupId,
        int generationId
    ) {
        ClassicGroup group;
        group = getOrMaybeCreateClassicGroup(groupId, false);

        if (group.isInState(COMPLETING_REBALANCE) || group.isInState(STABLE)) {
            // 如果还未接受到一些节点的 SyncGroupRequest
            if (!group.hasReceivedSyncFromAllMembers()) {
                Set<String> pendingSyncMembers = new HashSet<>(group.allPendingSyncMembers());
                // 移除这些 member，并取消对应的 heartbeat 探测任务
                pendingSyncMembers.forEach(memberId -> {
                    group.remove(memberId);
                    timer.cancel(classicGroupHeartbeatKey(group.groupId(), memberId));
                });

                // 重新发起 rebalance
                return prepareRebalance(group, "Removing " + pendingSyncMembers + " on pending sync request expiration");
            }
        }
        return EMPTY_RESULT;
    }
```

## 发生异常，怎么办

如果在重平衡的过程当中，发生了异常，怎么办？<br>
重新发起重平衡。

需要重新开始重平衡吗？如果需要，怎么通知消费者呢？通过 heartbeatResp 吗？<br>
是的通过 heartbeatResp。

消费者与 coordinator 之间的 heartbeat 又是什么时候开始的呢？<br>
实际上在 findCoordinator 成功后，便开始发送 heartbeat 消息。


### 如何区分不同轮的消息

实际上，在消费组的元信息中，维护了一个 generation_id 的字段，每当消费组发起重平衡的时候，该字段 + 1。当消费组 coordinator 接收到来自消费者的消息的时候，都会进行校验。如果 request 中的 generation_id <br>
与消费组元信息中的 generation_id 不相同，则直接返回错误。

我们以处理 SyncGroupRequest 为例，校验逻辑如下：

```java
private Optional<Errors> validateSyncGroup(
        ClassicGroup group,
        SyncGroupRequestData request
    ) {

        // 校验 generationId
        if (request.generationId() != group.generationId()) {
            return Optional.of(Errors.ILLEGAL_GENERATION);
        } else if (isProtocolInconsistent(request.protocolType(), group.protocolType().orElse(null)) ||
                    isProtocolInconsistent(request.protocolName(), group.protocolName().orElse(null))) {
            return Optional.of(Errors.INCONSISTENT_GROUP_PROTOCOL);
        } else {
            return Optional.empty();
        }
    }
```
# 几个重要的消费者参数

- session.timeout.ms

    如果超过这个时间还没有接收到 consumer 的 heartbeat，则 coordinator 认为 consumer 下线。

- heartbeat.interval.ms

    consumer 给 coordinator 发送心跳的时间间隔。

- max.poll.interval.ms

    consumer poll 消息的最大时间间隔，当 consumer 的两次 poll超过了这个时间段，则认为 consumer 下线。<br>
    当 consumer 的消费逻辑耗时很长或者遇到 gc 等场景下，可能遇到此问题。

- enable.auto.commit

    该参数控制 consumer 如何提交位移。无论如何设置，都有潜在的问题。

    - true

        当设置为 true 时，表示自动提交位移，如果某个消息消费失败了，但是位移自动提交了。这个消费就不能再消费，会导致数据的丢失。

    - false

        当设置为 false 时，表示需要手动提交。如果某个消息消费成功了，但是在提交位移前系统宕机了或者提交位移失败了，则会导致消息的重复消费。消费逻辑应该做好幂等处理。

# 其他

我们重点学习下 consumer group 中一些优秀的思路。

## coordinator

因为一个消费组中的消费者之间是不会进行通信的。那么当消费组中加入一个新的节点，其他节点如何感知这个信息并参与到重平衡流程呢？<br>
kafka 思路就是引入一个 coordinator 角色，消费者直接跟 coordinator 通信，那么 coordinator 又如何跟消费者通信呢？**通过 heartbeatResp 通信，这种思路也值得我们学习**。<br>
消费者无须提供 server 能力，而是不断通过 heartbeat 给 coordinator 上报自己的健康信息，而 coordinator 可以通过 heartbeatResp 告诉消费者一些重要的信息。

## 状态机

实际上，消费组跟 coordinator 的交互还是挺复杂的。kakfa 的思路是引入 `状态机` 来维护整个交互流程。<br>
这种状态机的思路值得我们学习，当涉及复杂的交互流程的时候，就可以使用状态机进行抽象。 