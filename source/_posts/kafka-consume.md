---
title: Kafka学习笔记 -- 读取数据
date: 2018-10-18 15:30:51
categories:
  - Kafka
tags:
  - Kafka
---

## 基本概念

### 消费者和消费者群组
**消费者从属于消费者群组**。一个群组里的消费者**订阅的是同一个主题**，**每个消费者接收主题一部分分区的消息**

#### 消费者横向扩展

##### 1个消费者
1. 主题T1有4个分区，然后创建消费者C1，C1是消费者群组G1里唯一的消费者，C1订阅T1
2. **消费者C1将接收主题T1的全部4个分区的消息**

![consumer_topic_1.png](http://pg67n0yz6.bkt.clouddn.com/consumer_topic_1.png?imageView2/2/w/300)

<!-- more -->

##### 2个消费者
1. 如果群组G1新增一个消费者C2，那么每个消费者将**分别从两个分区接收消息**
2. 假设C1接收分区0和分区2的消息，C2接收分区1和分区3的消息

![consumer_topic_2.png](http://pg67n0yz6.bkt.clouddn.com/consumer_topic_2.png?imageView2/2/w/300)

##### 4个消费者
1. 如果群组G1有4个消费者，那么每个消费者可以分配到一个分区

![consumer_topic_3.png](http://pg67n0yz6.bkt.clouddn.com/consumer_topic_3.png?imageView2/2/w/300)

##### 5个消费者
1. 如果群组G1有5个消费者，**超过主题的分区数量**，那么有1个消费者就会被**闲置**，不会接收到任何消息

![consumer_topic_4.png](http://pg67n0yz6.bkt.clouddn.com/consumer_topic_4.png?imageView2/2/w/300)

##### 总结
1. 往群组里增加消费者是**横向伸缩消费能力**的主要方式
2. 消费者经常会做一些高延迟的操作，比如把数据写到数据库或HDFS，或者使用数据进行比较耗时的计算
3. 有必要**为主题创建大量的分区**，在负载增长时可以加入更多的消费者
    - 不要让**消费者的数量超过主题分区的数量**，多余的消费者只会被**闲置**

#### 消费者群组横向扩展
1. Kafka设计的主要目标之一，就是要让Kafka主题里的数据能够满足企业各种应用场景（**不同的消费者群组**）的需求
2. 在这些场景里，每个应用程序可以获取到**所有的消息**，而不只是其中的一部分
3. 只要保证每个应用程序有自己的消费者群组，就可以让它们获取到主题所有的消息
4. 不同于传统的消息系统，**横向伸缩Kafka消费者和消费者群组并不会对性能造成负面影响**

![consumer_topic_5.png](http://pg67n0yz6.bkt.clouddn.com/consumer_topic_5.png?imageView2/2/w/300)

### 消费者群组+分区再均衡

#### 分区再均衡
1. 分区再均衡：**分区的所有权从一个消费者转移到另一个消费者**
2. 分区再均衡非常重要，它为消费者群组带来了**高可用性**和**伸缩性**（可以放心地添加或移出消费者）
3. 在分区再均衡期间，**消费者无法读取消息**，造成整个群组一小段时间内不可用
4. 当分区被重新分配给另一个消费者时，消费者当前的读取状态会丢失，它有可能需要去刷新缓存，在它重新恢复状态之前会拖慢应用程序

#### 心跳
1. 消费者通过向被指派为**群组协调器**的Broker（**不同的群组可以有不同的协调器**）发送**心跳**来维持它们**和群组的从属关系**以及它们**对分区的所有权关系**
2. 只要消费者以**正常的时间间隔**发送心跳，就被认为是**活跃**的，说明它还在读取分区里的消息
3. 消费者发送心跳的时机
    - **轮询消息**（为了获取消息）
    - **提交偏移量**
4. 如果消费者停止发送心跳的时间足够长，会话就会过期，群组协调器认为它已经死亡，就会触发一次再均衡
5. 如果一个消费者发生崩溃，并停止读取消息，群组协调器会等待几秒钟，确认它死亡了才会触发再均衡
    - 在这几秒的时间内，死掉的消费者不会读取分区里的消息
6. 在清理消费者时，消费者会**通知**协调器它将要离开群组，协调器会**立即**触发一次再均衡，尽量**降低处理停顿**

#### 分配分区
1. 当消费者要加入群组时，它会向**群组协调器**发送一个**JoinGroup的请求**，**第一个** 加入群组的消费者将成为**群主**
2. **群主** 从**协调器**那里获得**群组的成员列表**（列表中包含了**最近发送过心跳的消费者**，它们被认为是活跃的），**并负责给每个消费者分配分区**
    - 实现PartitionAssignor接口的类来决定哪些分区应该被分配给哪个消费者
3. 群主分配完毕后，把**分配情况列表**发送给**群组协调器**，**群组协调器再把这些信息发送给所有的消费者**
    - **每个消费者只能看到自己的分配信息，只有群主知道群组里所有消费者的分配信息**
4. 这个过程会在每次再均衡时重复发生

## 创建消费者
```java
Properties properties = new Properties();
properties.put("bootstrap.servers", "localhost:9092");
properties.put("group.id", "zhongmingmao");
properties.put("key.deserializer", StringDeserializer.class.getName());
properties.put("value.deserializer", StringDeserializer.class.getName());
Consumer<String, String> consumer = new KafkaConsumer<>(properties);
```

## 订阅主题
```java
// 订阅主题
// 支持正则表达式：如果创建新主题，并且主题的名字与正则表达式匹配，
//              就会触发一次再均衡，消费者就能读取新添加的主题
// consumer.subscribe("test.*");
consumer.subscribe(Collections.singletonList("CustomerCountry"));
```

## 轮询
1. 消息轮询是消费者API的核心，通过一个简单的轮询向服务器请求数据
2. 一旦消费者订阅了主题，轮询就会处理**所有的细节**
    - **群组协调**
    - **分区再均衡**
    - **发送心跳**
    - **获取数据**
3. 线程安全
    - 在同一个群组里，无法让一个线程运行多个消费者，也无法让多个线程安全地共享一个消费者
    - 按照规则，**一个消费者使用一个线程**

```java
try {
    while (true) {
        // 消费者必须持续对Kafka进行轮询，否则会被认为已经死亡，它的分区就会被移交给群组里的其它消费者
        // 在消费者的缓冲区里没有可用数据时会发生阻塞
        // 返回一个记录列表，每条记录包含
        //  1. 记录所属主题的信息
        //  2. 记录所在分区的信息
        //  3. 记录在分区里的偏移量
        //  4. 记录的键值对
        // timeout参数指定多久之后可以返回，不管有没有可用的数据，0会立即返回
        ConsumerRecords<String, String> records = consumer.poll(100);
        records.forEach(record -> log.info("topic={}, partition={}, offset={}, key={}, value={}",
                record.topic(), record.partition(), record.offset(), record.key(), record.value()));
    }
} finally {
    // 网络连接和Socket也会随之关闭
    // 并且立即触发一次再均衡，而不是等待群组协调器发现它不再发送心跳并认定它已死亡
    //  因为那样需要更长的时间，导致整个群组在一段时间内无法读取消息
    consumer.close();
}
```

## 消费者的配置

### fetch.min.bytes
1. 该参数指定了消费者从服务器获取记录的最小字节数
2. Broker在收到消费者的数据请求时，如果可用的数据量小于fetch.min.bytes指定的大小，那么它会等到有足够的可用数据时才会把它返回给消费者
    - 这样可以降低消费者和Broker的工作负载，因为在主题不是很活跃的时候，就不需要来回处理消息

### fetch.max.wait.ms
1. 如果fetch.max.wait.ms被设为100ms，并且fetch.min.bytes被设为1MB
    - 那么Kafka在收到消费者的请求后，要么返回1MB数据，要么在100ms后返回所有可用的数据，**看哪个条件先得到满足**

### max.partition.fetch.bytes
1. 该参数指定了服务器从**每个分区**里返回给消费者的最大字节数，默认1MB
    - KafkaConsumer.poll()方法从每个分区里返回的记录最多不超过max.partition.fetch.bytes
    - 如果单次poll()返回的数据太多，消费者需要更多时间来处理，可能无法及时进行下一轮询来避免会话过期
        - 减少max.partition.fetch.bytes的配置
        - 或者延长会话过期时间
2. 如果主题有20个分区和5个消费者，那么每个消费者需要至少4MB的可用内存来接收记录
    - 在为消费者分配内存时，可以适当地多分配些，因为如果群组里有消费者发生崩溃，剩下的消费者需要处理更多的分区
3. max.partition.fetch.bytes必须比**message.max.bytes**（Broker能够接收到最大消息的字节数）大
    - 否着消费者可能无法读取这些『大消息』，导致消费者一直挂起重试

### session.timeout.ms + heartbeat.interval.ms
1. session.timeout.ms指定了**消费者在被认为死亡之前可以与服务器断开连接的时间**，默认3s
    - 如果消费者没有在session.timeout.ms指定的时间内发送心跳给在**群组协调器**，就会被认为已经死亡，协调器会触发再均衡
2. heartbeat.interval.ms指定了poll()方法向协调器**发送心跳的频率**，而session.timeout.ms指定了消费者**可以多久不发心跳**
3. heartbeat.interval.ms必须比session.timeout.ms小，一般是session.timeout.ms的**1/3**
4. 如果把session.timeout.ms值设置得比默认值小，可以**更快地检测和恢复崩溃的节点**
    - 不过长时间的轮询或者GC可能会导致非预期的再均衡
5. 如果把session.timeout.ms值设置得比默认值大，**可以减少意外的再均衡**
    - 不过检测节点崩溃需要更长的时间

### auto.offset.reset
1. 该参数指定了消费者在读取一个**没有偏移量**（新加入消费者）的分区或者**偏移量无效**（因消费者长时间失效，包含偏移量的记录已经过时并被删除）的情况下，该如何处理
2. 默认值是**latest**：消费者从**最新**的记录开始读取数据
3. **earliest**：消费者从**起始位置**读取分区的记录

### enable.auto.commit + auto.commit.interval.ms
1. enable.auto.commit指定了消费者是否**自动提交偏移量**，默认值是true
    - 可能出现**重复数据**和**数据丢失**
2. auto.commit.interval.ms指定提交的**频率**

### partition.assignment.strategy
1. 分区会被分配给群组里的消费者
2. Range策略：把主题的若干**连续**的分区分配给消费者
3. RoundRobin策略：把主题的所有分区**逐个**分配给消费者

### client.id
1. 任意字符串，Broker用它来标识从客户端发过来的消息
2. 通常被用在日志、度量指标和配额里

### max.poll.records
1. 该参数用于控制单次调用poll()方法能够返回的记录数量

### receive.buffer.bytes + send.buffer.bytes
1. 设置socket在**读写数据**时用到的**TCP缓冲区**，如果为-1，就使用操作系统的默认值
2. 如果生产者或消费者与Broker处于不同的数据中心内，可以适当增大这些值

## 提交和偏移量
1. 每次调用poll()方法，总是返回由生产者写入Kafka但还没被消费者读取过的记录
2. **Kakfa不会像其它JMS队列那样需要得到消费者的确认**
3. 消费者可以使用Kafka来追踪消息在分区里的位置（偏移量）
4. 把**更新分区当前位置**的操作叫作**提交**
    - 消费者往一个叫作 **__consumer_offsets**的特殊主题发送消息，消息里包含每个**分区的偏移量**
    - 如果消费者发生崩溃或者有新的消费者加入群组，就会触发**再均衡**
    - 完成再均衡之后，每个消费者可能分配到新的分区，为了能继续之前都工作，消费者需要读取每个分区**最后一次提交的偏移量**，然后从偏移量指定的位置继续处理

如果提交的偏移量**小于**客户端处理的最后一个消息的偏移量，那么处于**两个偏移量之间的消息**就会被**重复处理**

![consumer_commit_1.png](http://pg67n0yz6.bkt.clouddn.com/consumer_commit_1.png?imageView2/2/w/400)

如果提交的偏移量**大于**客户端处理的最后一个消息的偏移量，那么处于**两个偏移量之间的消息**就会**丢失**

![consumer_commit_2.png](http://pg67n0yz6.bkt.clouddn.com/consumer_commit_2.png?imageView2/2/w/400)

### 自动提交
1. 自动提交是基于**时间间隔**
2. 如果enable.auto.commit为true，那么每过auto.commit.interval.ms（默认5s），消费者会自动把从poll()方法接收到的**最大偏移量**提交上去
3. 自动提交也在**轮询**里进行，消费者每次在进行轮询时会**检查是否该提交偏移量**，如果是，那就会提交**上一次**轮询返回的偏移量
4. **消息重复处理**
    - 假设使用5s的默认提交时间间隔，在最近一次提交之后的3s发生了再均衡
    - 再均衡之后，消费者从最后一次提交的偏移量位置开始读取消息
    - 这个时候偏移量已经落后了3s，所以在这3s内到达的消息会被重复处理
5. 使用自动提交，每次调用轮询方法都会把上一次调用返回的偏移量提交上去，**它并不知道具体哪些消息已经被处理过了**
    - 所以在再次调用轮询之前最好确保**所有当前调用返回的消息**都已经处理完毕

### 提交当前偏移量
1. enable.auto.commit=**false**，让应用程序决定何时提交偏移量
2. 使用commitAsync()提交偏移量**最简单也最可靠**
    - 提交由poll()方法返回的最新偏移量，提交成功后立马返回，如果提交失败就会抛出异常
3. 如果发生**再均衡**，从最近一批消息到发生再均衡之间的所有消息都将被**重复处理**

```java
consumer.subscribe(Collections.singletonList("CustomerCountry"));
try {
    while (true) {
        // 获取最新偏移量
        ConsumerRecords<String, String> records = consumer.poll(100);
        records.forEach(record -> log.info("topic={}, partition={}, offset={}, key={}, value={}",
                record.topic(), record.partition(), record.offset(), record.key(), record.value()));
        try {
            // 提交当前偏移量
            consumer.commitAsync();
        } catch (CommitFailedException e) {
            log.error("commit failed", e);
        }
    }
} finally {
    consumer.close();
}
```

### 异步提交
1. 手动提交，在Broker对提交请求作出响应之前，应用程序会一直**阻塞**，**限制应用程序的吞吐量**
    - 可以**降低提交频率**来**提升吞吐量**，但如果发生再均衡，会增加重复消息的数量
2. 在**成功提交**或碰到**无法恢复的错误**之前，commitSync()会一直重试，但是commitAsync()不会
    - 不进行重试的原因：在它收到服务器响应的时候，可能有一个**更大的偏移量**已经提交成功
3. commitAsync()支持回调，在Broker作出响应时会执行回调（通常用于记录提交错误或生成度量指标）
    - 如果用**回调**来进行**重试**，一定要注意**提交的顺序**
    - 使用一个**单调递增的序列号**来维护**异步提交的顺序**
    - 在每次提交偏移量之后或者在回调里提交偏移量时递增序列号
    - 在进行重试前，先检查**回调的序列号**和**即将提交到偏移量**是否相等，如果相等，说明没有新的提交，那么可以安全地进行重试
    - 如果序列号比较大，说明有一个新的提交已经发送出去了，应该停止重试

```java
consumer.subscribe(Collections.singletonList("CustomerCountry"));
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(100);
    for (ConsumerRecord<String, String> record : records) {
        log.info("topic={}, partition={}, offset={}, key={}, value={}",
                record.topic(), record.partition(), record.offset(),
                record.key(), record.value());
    }
    // 提交最后一个偏移量
    consumer.commitAsync((offsets, exception) -> {
        if (null != exception) {
            log.error("Commit failed for offsets " + offsets, exception);
        }
    });
}
```

### 组合提交（同步+异步）
1. 一般情况下，偶尔出现的提交失败，不进行重试不会有太大问题
2. 如果这是发生在**关闭消费者**或者**再均衡前**的**最后一次提交**，那么就要**确保能够提交成功**

```java
consumer.subscribe(Collections.singletonList("CustomerCountry"));
try {
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(100);
        for (ConsumerRecord<String, String> record : records) {
            log.info("topic={}, partition={}, offset={}, key={}, value={}",
                    record.topic(), record.partition(), record.offset(),
                    record.key(), record.value());
        }
        // 速度更快，如果这次提交失败，下一次提交很有可能会成功
        consumer.commitAsync((offsets, exception) -> {
            if (null != exception) {
                log.error("Commit failed for offsets " + offsets, exception);
            }
        });
    }
} catch (Exception e) {
    log.error("Unexpected error", e);
} finally {
    try {
        // 一直重试，直到提交成功或者发生无法恢复的错误
        consumer.commitSync();
    } finally {
        // 如果直接关闭消费者，就没有所谓的『下一次提交』了
        consumer.close();
    }
}
```

### 提交特定的偏移量
1. **提交偏移量的频率** 和**处理消息批次的频率**是一样的
2. commitSync()和commitAsync()，只会提交**最后一个偏移**，而此时该批次里的消息可能还没处理完
3. 如果需要提交特定的偏移量，需要**跟踪所有分区的偏移量**

```java
consumer.subscribe(Collections.singletonList("CustomerCountry"));
// 用于追踪偏移量的Map
Map<TopicPartition, OffsetAndMetadata> currentOffsets = Maps.newHashMap();
int count = 0;
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(100);
    for (ConsumerRecord<String, String> record : records) {
        log.info("topic={}, partition={}, offset={}, key={}, value={}",
                record.topic(), record.partition(), record.offset(),
                record.key(), record.value());
        // 在读取每条记录后，使用期望处理的下一条记录的偏移量更新map里的偏移量
        // 下一次就从这里开始读取消息
        currentOffsets.put(new TopicPartition(record.topic(), record.partition()),
                new OffsetAndMetadata(record.offset() + 1, "no metadata"));
        if (count++ % 1000 == 0) {
            // 每处理1000条记录就提交一次偏移量
            // 提交特定偏移量时，仍然要处理可能发生的错误
            consumer.commitAsync(currentOffsets, null);
        }
    }
}
```

## 再均衡监听器

## 从特定偏移量开始处理记录
<!-- indicate-the-source -->
