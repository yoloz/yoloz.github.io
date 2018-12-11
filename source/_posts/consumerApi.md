---
title: consumerApi
comments: false
toc: false
date: 2018-12-11 16:03:11
categories: kafka
tags:
---
如下是kafka消费者java api的订阅(subscribe)流程(v0.10)
<!--more-->

## initialize
实现类org.apache.kafka.clients.consumer.KafkaConsumer.java
``` java
private KafkaConsumer(ConsumerConfig config,
                      Deserializer<K> keyDeserializer,
                      Deserializer<V> valueDeserializer) {
    try {......} catch (Throwable t) {
        // call close methods if internal objects are already constructed
        // this is to prevent resource leak. see KAFKA-2121
        close(0, true);
        // now propagate the exception
        throw new KafkaException("Failed to construct kafka consumer", t);
    }
}
```
try块中，一些相关参数的处理(略)，其次一些对象的初始化。
``` java
ClusterResourceListeners clusterResourceListeners = configureClusterResourceListeners(keyDeserializer, valueDeserializer, reporters, interceptorList);
this.metadata = new Metadata(retryBackoffMs, config.getLong(ConsumerConfig.METADATA_MAX_AGE_CONFIG), false, clusterResourceListeners);
List<InetSocketAddress> addresses = ClientUtils.parseAndValidateAddresses(config.getList(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG));
this.metadata.update(Cluster.bootstrap(addresses), Collections.<String>emptySet(), 0);
```
> ClusterResourceListener  
A callback interface that users can implement when they wish to get notified about changes in the Cluster metadata. Users who need access to cluster metadata in interceptors, metric reporters, serializers and deserializers can implement this interface. 
回调接口，获取集群相关信息的变动。

> Metadata  
A class encapsulating some of the logic around metadata.
 This class is shared by the client thread (for partitioning) and the background sender thread.
 Metadata is maintained for only a subset of topics, which can be added to over time. When we request metadata for a topic we don't have any metadata for it will trigger a metadata update.
 If topic expiry is enabled for the metadata, any topic that has not been used within the expiry interval is removed from the metadata refresh set after an update. Consumers disable topic expiry since they explicitly manage topics while producers rely on topic expiry to limit the refresh set.  
元数据类。  
客户端线程(for partitioning)和后台发送线程共享元数据。  
元数据仅维护主题的一小部分，随时间增加，当向主题请求元数据时，如果没有元数据，则触发元数据更新。  
 如果元数据启用了到期功能，在更新操作中将从元数据集中删除过期的主题。消费者禁用主题到期，因为它们明确地管理主题，而生产者依靠主题到期来限制刷新集大小。

``` java
ChannelBuilder channelBuilder = ClientUtils.createChannelBuilder(config);
NetworkClient netClient = new NetworkClient(
        new Selector(config.getLong(ConsumerConfig.CONNECTIONS_MAX_IDLE_MS_CONFIG), metrics, time, metricGrpPrefix, channelBuilder),
        this.metadata,
       ................);
this.client = new ConsumerNetworkClient(netClient, metadata, time, retryBackoffMs,
        config.getInt(ConsumerConfig.REQUEST_TIMEOUT_MS_CONFIG));
```
> ChannelBuilder  
A ChannelBuilder interface to build Channel based on configs.

> NetworkClient  
A network client for asynchronous request/response network i/o. This is an internal class used to implement the user-facing producer and consumer clients.  
This class is not thread-safe!  
面向生产者消费者客户端的网络内部类，异步网络IO，非线程安全。

> ConsumerNetworkClient  
Higher level consumer access to the network layer with basic support for request futures. This class is thread-safe, but provides no synchronization for response callbacks. This guarantees that no locks are held when they are invoked.  
支持基本reuqest future的网络层，线程安全，但不提供响应回调的同步。可以确保在调用时不会保留锁。

``` java
OffsetResetStrategy offsetResetStrategy = OffsetResetStrategy.valueOf(config.getString(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG).toUpperCase(Locale.ROOT));
this.subscriptions = new SubscriptionState(offsetResetStrategy);
List<PartitionAssignor> assignors = config.getConfiguredInstances(
        ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
        PartitionAssignor.class);
this.coordinator = new ConsumerCoordinator(..............);
this.fetcher = new Fetcher<>(............................);
```
> OffsetResetStrategy  
枚举类LATEST, EARLIEST, NONE

> PartitionAssignor  
this interface is used to define custom partition assignment for use in KafkaConsumer. Members of the consumer group subscribe to the topics they are interested in and forward their subscriptions to a Kafka broker serving as the group coordinator. The coordinator selects one member to perform the group assignment and propagates the subscriptions of all members to it. Then assign(Cluster, Map)} is called to perform the assignment and the results are forwarded back to each respective members  
In some cases, it is useful to forward additional metadata to the assignor in order to make assignment decisions. For this, you can override {@link #subscription(Set)} and provide custom userData in the returned Subscription. For example, to have a rack-aware assignor, an implementation can use this user data to forward the rackId belonging to each member.  
消费者的分区分配接口

> ConsumerCoordinator  
This class manages the coordination process with the consumer coordinator.  
消费者协调员管理

> Fetcher  
This class manage the fetching process with the brokers.  
管理从broker抓取数据

## subscribe
``` java
public void subscribe(Collection<String> topics) {
    subscribe(topics, new NoOpConsumerRebalanceListener());
}
```
Subscribe to the given list of topics to get dynamically assigned partitions.Topic subscriptions are not incremental. This list will replace the current assignment (if there is one).It is not possible to combine topic subscription with group management with manual partition assignment through assign(Collection).  
订阅给定主题列表，分区动态分配。与自定义分配(assign())不可同时使用。
``` java
public void subscribe(Collection<String> topics, ConsumerRebalanceListener listener) {
.......................
     this.subscriptions.subscribe(new HashSet<>(topics), listener);
     metadata.setTopics(subscriptions.groupSubscription());
.......................
}
```
As part of group management, the consumer will keep track of the list of consumers that belong to a particular group and will trigger a rebalance operation if one of the following events trigger  
新消费者的加入，组管理中消费者会可能属于已经存在的组，或者新组。如果发生以下事件将触发重新平衡操作：
* Number of partitions change for any of the subscribed list of topics.已订阅主题列表的分区改动
* Topic is created or deleted.主题创建或删除
* An existing member of the consumer group dies.组中有成员离线(死亡)
* A new member is added to an existing consumer group via the join API.组中有新成员加入

当触发任何这些事件时，将首先调用提供的侦听器来指示消费者的作业已被撤销，然后再次接收到新的作业。

## consumer poll
实现类org.apache.kafka.clients.consumer.KafkaConsumer.java
``` java
public ConsumerRecords<K, V> poll(long timeout) {
    acquire();
    try {
        if (timeout < 0)
            throw new IllegalArgumentException("Timeout must not be negative");

        if (this.subscriptions.hasNoSubscriptionOrUserAssignment())
            throw new IllegalStateException("Consumer is not subscribed to any topics or assigned any partitions");

        // poll for new data until the timeout expires
        long start = time.milliseconds();
        long remaining = timeout;
        do {
         Map<TopicPartition, List<ConsumerRecord<K, V>>> records =pollOnce(remaining);
            if (!records.isEmpty()) {
                  if (fetcher.sendFetches() > 0 ||
        client.hasPendingRequests()){client.pollNoWakeup();}
.......}
            long elapsed = time.milliseconds() - start;
            remaining = timeout - elapsed;
        } while (remaining > 0);
        return ConsumerRecords.empty();
} finally {
        release();
    }
}
```
> acquire()  
判断consumer是否多线程访问，consumer非线程安全，如果多线程使用会报异常ConcurrentModificationException

``` java
if(!records.isEmpty()){
    if(fetcher.sendFetches()>0||client.hasPendingRequests()){
        client.pollNoWakeup();
    }
    ........
}
```
before returning the fetched records, we can send off the next round of fetches and avoid block waiting for their responses to enable pipelining while the user is handling the fetched records.  
在返回抓取的数据时，发送下一轮提取请求。即，用户在接收到数据的时候，下一次提取数据的请求已经发出去了。

> pollOnce  
实现类org.apache.kafka.clients.consumer.KafkaConsumer.java  
Do one round of polling. In addition to checking for new data, this does any needed offset commits (if auto-commit is enabled), and offset resets (if an offset reset policy is defined).  
一次poll。除了检查新数据，还会涉及offset提交以及offset reset。
``` java
private Map<TopicPartition, List<ConsumerRecord<K, V>>> pollOnce(long timeout) {
    client.maybeTriggerWakeup();
    coordinator.poll(time.milliseconds());
    // fetch positions if we have partitions we're subscribed to that we
    // don't know the offset for
    if (!subscriptions.hasAllFetchPositions())
        updateFetchPositions(this.subscriptions.missingFetchPositions());
    // if data is available already, return it immediately
   Map<TopicPartition, List<ConsumerRecord<K, V>>> records = fetcher.fetchedRecords();
    if (!records.isEmpty())
        return records;
    // send any new fetches (won't resend pending fetches)
    fetcher.sendFetches();
    long now = time.milliseconds();
    long pollTimeout = Math.min(coordinator.timeToNextPoll(now), timeout);
    client.poll(pollTimeout, now, new PollCondition() {
        @Override
        public boolean shouldBlock() {
       // since a fetch might be completed by the background thread, we need this poll condition to ensure that we do not block unnecessarily in poll()
            return !fetcher.hasCompletedFetches();
        }
    });
    // after the long poll, we should check whether the group needs to rebalance prior to returning data so that the group can stabilize faster
    if (coordinator.needRejoin())
        return Collections.emptyMap();
    return fetcher.fetchedRecords();
}
```
> client.maybeTriggerWakeup()  
响应用户的wakeup，抛出WakeupException，如多线程使用中以此来停止consumer。  

> coordinator.poll  
实现类org.apache.kafka.clients.consumer.internals.ConsumerCoordinator.java  
Poll for coordinator events. This ensures that the coordinator is known and that the consumer has joined the group (if it is using group management). This also handles periodic offset commits if they are enabled.  
coordinator的poll事件，如下：
``` java
public void poll(long now) {
    invokeCompletedOffsetCommitCallbacks();
    if (subscriptions.partitionsAutoAssigned() && coordinatorUnknown()) {
        ensureCoordinatorReady();
        now = time.milliseconds();
    }
    if (needRejoin()) {
// due to a race condition between the initial metadata fetch and the initial rebalance, we need to ensure that the metadata is fresh before joining initially. This ensures that we have matched the pattern against the cluster's topics at least once before joining.
        if (subscriptions.hasPatternSubscription())
            client.ensureFreshMetadata();
            ensureActiveGroup();
            now = time.milliseconds();
    }
    pollHeartbeat(now);
    maybeAutoCommitOffsetsAsync(now);
}
```
> invokeCompletedOffsetCommitCallbacks  
ConcurrentLinkedQueue<OffsetCommitCompletion>：
this collection must be thread-safe because it is modified from the response handler of offset commit requests, which may be invoked from the heartbeat thread.  
线程安全的集合(队列)，commit offset请求响应逻辑中更改或者心跳线程中调用。offsetcommit的callback返回中处理一些逻辑，如果已消费的offset记录server没有表示，得阻塞处理(invoke)。

``` java
if (subscriptions.partitionsAutoAssigned() && coordinatorUnknown()) {
        ensureCoordinatorReady();
        now = time.milliseconds();
    }
```
> 保证coordinator就绪(是好的)，否则阻塞(ensureCoordinatorReady()阻塞处理逻辑)  
Block until the coordinator for this group is known and is ready to receive requests.

> needRejoin()  
``` java
if (!subscriptions.partitionsAutoAssigned())
    return false;
// we need to rejoin if we performed the assignment and metadata has changed
if (assignmentSnapshot != null && !assignmentSnapshot.equals(metadataSnapshot))
    return true;
// we need to join if our subscription has changed since the last join
if (joinedSubscription != null && !joinedSubscription.equals(subscriptions.subscription()))
    return true;
return super.needRejoin();
```

如果调用subscribe接口消费，则partitionsAutoAssigned为true。  
如果使用模式匹配的话，则要ensureFreshMetadata：
Ensure our metadata is fresh (if an update is expected, this will block until it has completed).

> ensureActiveGroup()  
Ensure that the group is active (i.e. joined and synced)
``` java
public void ensureActiveGroup() {
// always ensure that the coordinator is ready because we may have been disconnected when sending heartbeats and does not necessarily require us to rejoin the group.
    ensureCoordinatorReady();
    startHeartbeatThreadIfNeeded();
    joinGroupIfNeeded();
}
```

> ensureCoordinatorReady()  
ensureCoordinatorReady():Block until the coordinator for this group is known and is ready to receive requests.

>  pollHeartbeat(now)  
Check the status of the heartbeat thread (if it is active) and indicate the liveness of the client. This must be called periodically after joining with ensureActiveGroup() to ensure that the member stays in the group. If an interval of time longer than the provided rebalance timeout expires without calling this method, then the client will proactively leave the group.  
通过心跳判断client是否存活，周期性的发送心跳包，如果超时则client将被移出所在组。

>  maybeAutoCommitOffsetsAsync(now)  
``` java
private void maybeAutoCommitOffsetsAsync(long now) {
    if (autoCommitEnabled) {
        if (coordinatorUnknown()) {
            this.nextAutoCommitDeadline = now + retryBackoffMs;
        } else if (now >= nextAutoCommitDeadline) {
            this.nextAutoCommitDeadline = now + autoCommitIntervalMs;
            doAutoCommitOffsetsAsync();
        }
    }
}

private void doAutoCommitOffsetsAsync() {
Map<TopicPartition, OffsetAndMetadata> allConsumedOffsets = subscriptions.allConsumed();
    commitOffsetsAsync(allConsumedOffsets, new OffsetCommitCallback() {......});
}

public void commitOffsetsAsync(final Map<TopicPartition, OffsetAndMetadata> offsets, final OffsetCommitCallback callback) {
invokeCompletedOffsetCommitCallbacks();
if (!coordinatorUnknown()) {
   doCommitOffsetsAsync(offsets, callback);
    } else {.......}

private void doCommitOffsetsAsync(final Map<TopicPartition, OffsetAndMetadata> offsets, final OffsetCommitCallback callback) {
    this.subscriptions.needRefreshCommits();
    RequestFuture<Void> future = sendOffsetCommitRequest(offsets);
...................
    future.addListener(new RequestFutureListener<Void>() {
..............
completedOffsetCommits.add(new OffsetCommitCompletion(cb, offsets, null));
..............
}
```
> sendOffsetCommitRequest(offsets)  
Commit offsets for the specified list of topics and partitions. This is a non-blocking call which returns a request future that can be polled in the case of a synchronous commit or ignored in the asynchronous case.  
提交具体的topic和partition的offsets，非阻塞，返回future.

> subscriptions.allConsumed()  
获取提交的offset，如下：
``` java
public Map<TopicPartition, OffsetAndMetadata> allConsumed() {
......................   
allConsumed.put(state.topicPartition(),newOffsetAndMetadata(state.value().position).............}
//state.value().position即下面的position，最后消费的位置，由下文的fetchPositions更新

private static class TopicPartitionState {
    private Long position; // last consumed position
    private Long highWatermark; // the high watermark from last fetch
    private OffsetAndMetadata committed;  // last committed position
    private boolean paused;  // whether this partition has been paused by the user
    private OffsetResetStrategy resetStrategy;  // the strategy to use if the offset needs resetting
```

> updateFetchPositions()  
fetch positions if we have partitions we're subscribed to that we don't know the offset for  
判断所有partition的offset是否有效，否则更新。
``` java
 if (!subscriptions.hasAllFetchPositions())
        updateFetchPositions

private void updateFetchPositions(Set<TopicPartition> partitions) {
    // lookup any positions for partitions which are awaiting reset 
fetcher.resetOffsetsIfNeeded(partitions);
    if (!subscriptions.hasAllFetchPositions(partitions)) {
// if we still don't have offsets for the given partitions, then we should either eek to the last committed position or reset using the auto reset policy
    // first refresh commits for all assigned partitions
      coordinator.refreshCommittedOffsetsIfNeeded();
    // then do any offset lookups in case some positions are not known
        fetcher.updateFetchPositions(partitions);
    }
}
```

> fetchedRecords()  
Return the fetched records, empty the record buffer and update the consumed position.  
NOTE: returning empty records guarantees the consumed position are NOT updated.  
返回获取的记录，清空记录缓冲区并更新消费的位置(consumed position)  
注意：返回空记录保证消费的位置不更新
``` java
public Map<TopicPartition, List<ConsumerRecord<K, V>>> fetchedRecords() {
  ..........
while (recordsRemaining > 0) {
      ................
List<ConsumerRecord<K, V>> records=fetchRecords(nextInLineRecords,recordsRemaining);
if (!records.isEmpty()) {
      List<ConsumerRecord<K, V>> currentRecords = fetched.get(partition);
          if (currentRecords == null) {
                    fetched.put(partition, records);
          } else {
 // this case shouldn't usually happen because we only send one fetch at a time per partition, but it might conceivably happen in some rare cases (such as partition leader changes). we have to copy to a new list because the old one may be immutable
List<ConsumerRecord<K, V>> newRecords = new ArrayList<>(records.size() + currentRecords.size());
                    newRecords.addAll(currentRecords);
                    newRecords.addAll(records);
                    fetched.put(partition, newRecords);
                }
                recordsRemaining -= records.size();
            }
        }
    }

    return fetched;
}
```

> fetchRecords(nextInLineRecords,recordsRemaining)  
``` java
private List<ConsumerRecord<K, V>> fetchRecords(PartitionRecords partitionRecords, int maxRecords) {
   ..................
      long position = subscriptions.position(partitionRecords.partition);
   ....................   
   if (partitionRecords.nextFetchOffset == position) {
List<ConsumerRecord<K, V>> partRecords = partitionRecords.fetchRecords(maxRecords);
  long nextOffset = partitionRecords.nextFetchOffset;
  //更新consumed position
  subscriptions.position(partitionRecords.partition, nextOffset); 
...........................
}
```

> client.poll(pollTimeout, now, new PollCondition()...)  
Do actual reads and writes from sockets.

> sendFetches()  
Set-up a fetch request for any node that we have assigned partitions for which doesn't already have an in-flight fetch or pending fetch data.  
发送新的抓取请求(不会发送等待处理的请求)。如未发送过fetch请求的分区节点或挂起数据的分区节点。partition的读写请求落于leader,leader落于一台broker。一个partition一个fetch请求。
``` java
public int sendFetches() {
    Map<Node, FetchRequest.Builder> fetchRequestMap = createFetchRequests();
    for (Map.Entry<Node, FetchRequest.Builder> fetchEntry : fetchRequestMap.entrySet()) {
        final FetchRequest.Builder request = fetchEntry.getValue();
        final Node fetchTarget = fetchEntry.getKey();
        client.send(fetchTarget, request)
                .addListener(new RequestFutureListener<ClientResponse>() {......});
}
    return fetchRequestMap.size();
}
```

> createFetchRequests()  
Create fetch requests for all nodes for which we have assigned partitions that have no existing requests in flight.  
创建fetch request
``` java
private Map<Node, FetchRequest.Builder> createFetchRequests() {
    // create the fetch info
   .............
for (TopicPartition partition : fetchablePartitions()) {
.............
LinkedHashMap<TopicPartition, FetchRequest.PartitionData> fetch.......
fetchable.put(node, fetch);
.............
long position = this.subscriptions.position(partition);
//fetch中放入此partition的last consumed position
fetch.put(partition, new FetchRequest.PartitionData(position, FetchRequest.INVALID_LOG_START_OFFSET, this.fetchSize));
................
}
// create the fetches
Map<Node, FetchRequest.Builder> requests = new HashMap<>();
for (Map.Entry<Node, LinkedHashMap<TopicPartition, FetchRequest.PartitionData>> entry : fetchable.entrySet()) {
Node node = entry.getKey();
FetchRequest.Builder fetch = FetchRequest.Builder.forConsumer(this.maxWaitMs, this.minBytes, entry.getValue()).setMaxBytes(this.maxBytes);
        requests.put(node, fetch);
}
return requests;}
```

> forConsumer(this.maxWaitMs...)  
``` java
public static Builder forConsumer(int maxWait, int minBytes, LinkedHashMap<TopicPartition, PartitionData> fetchData) {
    return new Builder(null, CONSUMER_REPLICA_ID, maxWait, minBytes, fetchData);
}
```
