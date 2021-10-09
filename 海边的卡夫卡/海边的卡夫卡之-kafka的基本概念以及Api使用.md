# 海边的卡夫卡之 - kafka的基本概念以及Api使用

## kafka的应用以及与其他MQ的对比

关于kafka的介绍，也许没有人能比官网更具有话语权，所以这里可以参考官网了解一下kafka：[Kafka介绍](http://kafka.apache.org/)。这里从一下几个方面稍微总结一下：

**kafka的核心能力：**

- 高吞吐量：RabbitMq、RocketMq和Kafka中，吞吐量最高的就是Kafka
- 可扩展：生产集群可以弹性扩展到至1000多个broker，数十万个分区，每天处理万亿条消息，PB级数据，
- 持久化：每个topic可以有多个分区，每个分区又可以有多个副本，数据可以安全持久的存储在分布式的集群中，kafka中的消息消费后不会删除
- 高可用：支持集群高可用

**Kafka可以用来做什么**

- 消息中间件的解耦、削峰、异步
- 日志收集，可以用kafka收集各种log，然后统一提供给其他的消费者

**Kafka和其他MQ的对比**

关于这个问题这里有一篇文章写的不错，分享给大家：[RabbitMQ在国内为什么没有那么流行？](https://www.zhihu.com/question/449611434/answer/1824707689)。我给大家在总结一下

1. Kafka VS ActiveMQ  ActiveMQ已经属于上一代MQ了，性能差，重量级，以及退出了历史舞台
2. Kafka VS RabbitMQ RabbitMQ相对来说性能比ActiveMQ高很多，但远远不如Kafka，但是他的消息可靠性好，常用在金融领域，并且他是erlang语言写的，懂得人少
3. Kafka VS RocketMQ RocketMQ相当于是用java改写了Kafka，他的性能比RabbitMQ高，但还是比不过Kafka。RocketMQ社区活跃，功能服务，支持事务消息、顺序消息、私信队列等。kafka就没有这么多特性，但是他的吞吐来量是最高的，因此常用在大数据领域，日志收集等方面。

## kafka的基本概念

每一台部署了Kafka的机器，可以被称作一个**broker**，多个broker可以组成一个kafka的集群。broker用来保存处理**message(消息)**。在Kafka的定义中，消息也可以称作**event(事件)或者record(记录)**。每一条消息都可以抽象出三个属性：**key**、**value**和**timestamp**。而这些消息的来源被叫做**Producers**，Producers生产消息并将它们发布写入到Kafka的broker中。然后**Consumers**会订阅和消费这些消息。

实际业务中，有成千上万的消息，他们有的也许记录的天气信息，有的也许记录的是物流信息，有的也许记录的是日志信息，那么如果Producers把这些消息不加区分都发给Broker，则给Consumer的消费带来了巨大的挑战，因为Consumer不知道他们应该拉取哪些消息，哪些应该是他关注的。因此Kafka提出了一个**Topic**的概念，用来对消息进行分类存储。Topic就类似一个文件夹，比如天气预报这个文件夹下面就只存储与天气有关的文件。有了Topic这个概念后，Consumer就可以根据Topic进行消息的精准定位拉取，不会乱。

而且Topic还存在一个**partition**的概念。也就是说每一个Topic可以有多个partition，而且这个partition可能分布在不同的Broker节点上，就好像我们天气预报这个文件夹，可能因为文件太多，而不能放在一个硬盘上，我们可以将他分成几个子文件夹，分别存储在多台电脑上。这种设计方式很大程度上提高了他的可扩展性，因为他可以实现多个客户端同时从Broker读取或者写入消息。

为了保证高可用，Topic还有一个**replicate**的概念，也就是说每一个Topic都可以有好几个备份的副本，存储在不同的Broker上。其中一个会被选为主副本，其他的都是从副本，而每次有消息写入的时候，都只会写到主副本上，然后同步给其他的副本，当其中主副本宕机后，kafka会选举出一个新的副本提供服务。这样一来，假设某一天有一个节点挂了，那么也不用担心数据丢失或者不能正常提供服务。

接下来我们创建一个三个节点的Kafka集群，来验证一下上面的理论知识。因为Kafka目前是强依赖zookeeper做注册中心的，所以第一步要先安装zookeepr，这里省去不提，对Kafka的安装，也省去不提，可参考官网的[quick start](http://kafka.apache.org/quickstart)。安装完成后，进入到Kafka的`config`目录下，将`server.properties`拷贝三份，修改启动端口号和日志文件路径，模拟一个虚拟的分布式集群架构。然后通过如下命令：`bin/kafka-topics.sh --create --zookeeper 192.168.0.108:2181 --replication-factor 3 --partitions 2 --topic test`创建一个有两个分区，三个副本的名为`test`的Topic。然后通过如下命令`bin/kafka-topics.sh --describe --zookeeper 192.168.0.108:2181 --topic test`可查看`test`的配置。

```
# 总体描述，两个分区，三个副本
Topic: test	PartitionCount: 2	ReplicationFactor: 3	Configs:
	# 分区0: Leader是0,(0是Broker的Id,每个Broker有一个唯一ID), 副本在0,1,2三个节点上, Isr列出了存活的以及做了数据同步的节点
	Topic: test	 Partition: 0	Leader: 0	Replicas: 0,1,2	Isr: 0,1,2 
	Topic: test	 Partition: 1	Leader: 1	Replicas: 0,1,2	Isr: 1,0,2
```

通过查看配置验证了以上的说法以后，在通过一张图总结一下：

![](img\Kafka架构图.png)

## Kafka消费消息的两种模式

### 队列模式

在实际开发中，可能会遇到这样的场景，服务端有很多消息，那如果只有一个consumer可能处理不过来，所以我们会加几个consumer一起处理，但是一个消息只会被一个Consumer处理，这个在RabbitMQ里叫工作队列模式，做法是让多个消费者绑定到一个队列，然后MQ会派发消息。那么在Kafka里消息是发送到Topic的partition上的，而consumer是被Consumer Group组织和管理的。所以Kafka实现这种的模式的方式就是让多个Consumer属于同一个Consumer Group，每一个Consumer负责拉去一个partition的消息处理。那么如果有四个partition，但是又5个Consumer会怎么样呢？答案就是多余的一个啥都干不了，浪费。

### 发布订阅模式

发布订阅模式应该都很熟，不管是RabbitMQ的发布订阅模式、ActiveMQ的点对多模式等，都是发布订阅模式。他和对列模式的区别就是，发布一条消息要让多个消费者同时消费。那么结合队列模式思考，他的实现就很简单了，那就是让这几个Consumer属于不同的Consumer Group即可。

## Topic分区的结构

了解了整体架构，还有必要了解一下消息是如何存储的。假设我们创建了一个名为：replicated-system-log的两分区三副本的Topic，我们去看一下其中一个节点的日志目录下会有什么呢？我们发现会有两个文件夹，后缀加了一个0和1，代表了分区0和分区1。

![](img\两个文件夹.png)

然后在选一个文件夹进去，其中`00000000000000000000.log`负责保存消息，其他两个负责存储索引，暂且不表，下回分解。

![](img\commit log.png)

下图可以看作是对commit log的一个详解，可以看作是一个队列，每一次消息写入的时候会为递增的分配一个编号确保唯一性，我们称之为`offset`。当一个commit log写完后，会重新在创建一个commit log。而命名就是上一次结束的offset。假设默认的是`00000000000000000000.log`，当他被写满后，最后一个消息的offset为12，那么下一个commit log文件名称就是`00000000000000000012.log`。

Consumer消费消息也是以这个offset来进行的。每次消费完Consumer都会维护新的offset，表示一个进度，下次消费的时候就会接着这个offset往后消费。因为Kafka不会删除消息，因此也可以指定消费某分区某offset的消息，或之后的消息，或一段时间的消息，或回溯消费所有消息。

![](img\topic分区.png)

## kafka的java客户端使用

### Producer如何发消息

代码中所有的配置参考：[3.3 Producer Configs](http://kafka.apache.org/24/documentation.html#producerconfigs)

```java
/**
 * 生产者的一个基类，设置了一些基础参数
 */
public abstract class BaseProducer {

    public static final Producer<String, String> PRODUCER;

    static {
        Properties props = new Properties();
        // kafka的服务器配置
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, SERVER_ADDRESS);
        
        // 1. acks = 0, 这种模式producer只负责发消息，不等待broker确认是否收到，性能很高，但容易产生丢消息，
        //   而且这种模式的话，重试机制也会失效
        // 2. acks = 1, 这种模式，producer会等待leader节点确认真的收到消息了，也写到日志了，但是他不等待follower的确认
        //   这样的话，如果leader收到消息后立刻宕机，会发生丢消息
        // 3. acks = -1或all, 这种模式，producer不仅等leader确认，还会等所有副本都确认收到消息才算OK，可靠性最高，但效率不高
        props.put(ProducerConfig.ACKS_CONFIG, "1");
        
        // 发送失败会重试可以重试几次
        props.put(ProducerConfig.RETRIES_CONFIG, 3);
        
        // 重试间隔
        props.put(ProducerConfig.RETRY_BACKOFF_MS_CONFIG, 300);
        
        // 本地缓冲区，kafka会将消息先发送到本地缓冲区来提高消息发送性能，默认值是33554432字节(32MB)
        props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 33554432);
        
        // kafka会有一个线程负责从缓冲区取出数据批量发送，默认满16384字节(16KB)就发送
        props.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);
        
        // 那如果只发了一条消息，一时半会到不了16KB怎么办，还有一个参数用来设置如果到了规定的时间还没满16KB，就将消息发出去
        props.put(ProducerConfig.LINGER_MS_CONFIG, 10);
        
        // 序列化
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

        PRODUCER = new KafkaProducer<String, String>(props);
    }

    /**
     * 生产者发送消息
     */
    public abstract void sendMessage() throws ExecutionException, InterruptedException;
}
```

#### 同步发送消息

```java
/**
 * 同步发送消息
 * 
 * @author : HXY
 * @date : 2021-06-05 01:05
 **/
@Slf4j
public class SyncProducer extends BaseProducer{

    public void sendMessage() throws ExecutionException, InterruptedException {
        DateFormat df = DateFormat.getTimeInstance(DateFormat.MEDIUM, Locale.CHINA);
        String[] logLevel = new String[] {"DEBUG", "INFO", "WARN", "ERROR"};

        for (int i = 0; i < 10; i++) {
            LogBean logBean = new LogBean(i, df.format(new Date()), logLevel[i / 4], "log content" + i);

            // 1. 构建producerRecord对象
            ProducerRecord<String, String> producerRecord =
                    new ProducerRecord<>(TOPIC_NAME, logBean.getId().toString(), JSON.toJSONString(logBean));
            // 2. 发送消息，同步等待获取结果
            RecordMetadata recordMetadata = PRODUCER.send(producerRecord).get();
            log.info("sync send result success：topic : {}, partition : {}, offset : {}", recordMetadata.topic(), recordMetadata.partition(), recordMetadata.offset());
        }
        PRODUCER.close();
    }
}
```

#### 异步发送消息

```java
/**
 * 异步发送消息
 * 
 * @author : HXY
 * @date : 2021-06-05 01:05
 **/
@Slf4j
public class AsyncProducer extends BaseProducer{

    public void sendMessage() throws ExecutionException, InterruptedException {
        DateFormat df = DateFormat.getTimeInstance(DateFormat.MEDIUM, Locale.CHINA);
        String[] logLevel = new String[] {"DEBUG", "INFO", "WARN", "ERROR"};
        final CountDownLatch latch = new CountDownLatch(10);

        for (int i = 0; i < 10; i++) {
            LogBean logBean = new LogBean(i, df.format(new Date()), logLevel[i / 4], "log content" + i);

            // 1. 构建producerRecord对象
            ProducerRecord<String, String> producerRecord =
                    new ProducerRecord<>(TOPIC_NAME, logBean.getId().toString(), JSON.toJSONString(logBean));
            // 2. 异步发送消息，在回调函数中获取发送结果
            PRODUCER.send(producerRecord, (recordMetadata, e) -> {
                if (null != recordMetadata) {
                    log.info("async send result success：topic : {}, partition : {}, offset : {}", recordMetadata.topic(), recordMetadata.partition(), recordMetadata.offset());
                }
                if (null != e) {
                    log.error("async send result topic failed...", e);
                }
                latch.countDown();
            });
        }
        latch.await();
        PRODUCER.close();
    }
}
```

### Consumer如何拉消息

封装了Consumer的基本配置，参考[3.4 Consumer Configs](http://kafka.apache.org/24/documentation.html#consumerconfigs)

```java
public abstract class BaseConsumer {

    public KafkaConsumer<String, String> buildConsumer(Properties props) {
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, SERVER_ADDRESS);
        // 消费分组名
        props.put(ConsumerConfig.GROUP_ID_CONFIG, CONSUMER_GROUP_NAME);

        // consumer给broker发送心跳的间隔时间，broker接收到心跳如果此时有rebalance发生会通过心跳响应将
        // rebalance方案下发给consumer，这个时间可以稍微短一点
        props.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, 1000);

        // 超过了这个时间，broker没有收到consumer的心跳，则会将他淘汰
        // 对应的分区也会被重新分配给其他consumer，默认是10秒
        props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 10 * 1000);

        // 一次最大拉取消息的条数
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 100);

        // 如果超过了这个时间没有在调用poll()方法拉取消息，那么这个消费者也将被淘汰，对应的分区会分配给别的consumer
        props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 30 * 1000);

        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());

        return new KafkaConsumer<>(props);
    }

    public abstract void consume();
}
```

#### 自动提交offset

```java
public class AutoCommitConsumer extends BaseConsumer {

    /**
     * 自动提交offset，容易导致消息丢失和重复消费的问题，假设auto.commit.interval.ms设置了1秒
     * 1. 消息丢失：每隔1秒提交一次offset，但如果消费者消费能力不行，需要2秒，并且在1.5秒的时候宕机了，那么就会造成消息丢失
     * 2. 重复消费：假设0.5秒就消费完了所有消息，但是offset未提交，服务宕机了，那么就会导致重复消费
     */
    @Override
    public void consume() {
        Properties props = new Properties();
        // offset提交策略，默认为true，后台会定期默认提交
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "true");
        // 在上一个参数设置为true的情况下生效，指定提交offset的频率，以毫秒为单位，默认5000
        props.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, "1000");

        KafkaConsumer<String, String> consumer = buildConsumer(props);

        consumer.subscribe(Collections.singletonList(TOPIC_NAME));
        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
            for (ConsumerRecord<String, String> record : records) {
                log.info("receive message：partition : {}, offset : {}, content : {}", record.partition(), record.offset(), record.value());
            }
        }
    }
}
```

#### 手动提交offset

```java
@Slf4j
public class ManualCommitConsumer extends BaseConsumer{

    @Override
    public void consume() {
        Properties props = new Properties();
        // 手动提交offset
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");

        KafkaConsumer<String, String> consumer = buildConsumer(props);

        consumer.subscribe(Collections.singletonList(TOPIC_NAME));
        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
            for (ConsumerRecord<String, String> record : records) {
                log.info("receive message：partition : {}, offset : {}, content : {}", record.partition(), record.offset(), record.value());
            }

            if (records.count() > 0) {
                // 手动同步提交offset
                // consumer.commitSync();

                // 手动异步提交offset
                consumer.commitAsync((offsets, exception) -> {
                    if (exception != null) {
                        log.error("{} commit failed...", offsets);
                        log.error("commit failed exception: " + exception);
                    }
                });

            }
        }
    }
}
```

#### 从指定offset消费，拉去之后的消息

```java
@Slf4j
public class OffsetConsumer extends BaseConsumer{

    @Override
    public void consume() {
        Properties props = new Properties();
        // 手动提交offset
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");

        KafkaConsumer<String, String> consumer = buildConsumer(props);

        // 从offset = 5开始消费后面的数据
        List<TopicPartition> topicPartitions = Arrays.asList(new TopicPartition(TOPIC_NAME, 0), new TopicPartition(TOPIC_NAME, 1));
        consumer.assign(topicPartitions);
        consumer.seek(new TopicPartition(TOPIC_NAME, 0), 90);

        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
            for (ConsumerRecord<String, String> record : records) {
                log.info("receive message：partition : {}, offset : {}, content : {}", record.partition(), record.offset(), record.value());
            }

            if (records.count() > 0) {
                consumer.commitAsync((offsets, exception) -> {
                    if (exception != null) {
                        log.error("{} commit failed...", offsets);
                        log.error("commit failed exception: " + exception);
                    }
                });

            }
        }
    }
}
```

#### 消费指定分区

```java
@Slf4j
public class PartitionConsumer extends BaseConsumer{

    @Override
    public void consume() {
        Properties props = new Properties();
        // 手动提交offset
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
        KafkaConsumer<String, String> consumer = buildConsumer(props);
        
        // 指定消费分区1的消息
        consumer.assign(Collections.singletonList(new TopicPartition(TOPIC_NAME, 1)));
        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
            for (ConsumerRecord<String, String> record : records) {
                log.info("receive message：partition : {}, offset : {}, content : {}", record.partition(), record.offset(), record.value());
            }

            if (records.count() > 0) {
                consumer.commitAsync((offsets, exception) -> {
                    if (exception != null) {
                        log.error("{} commit failed...", offsets);
                        log.error("commit failed exception: " + exception);
                    }
                });

            }
        }
    }

    public static void main(String[] args) {
        PartitionConsumer consumer = new PartitionConsumer();
        consumer.consume();
    }
}
```

#### 消费所有历史消息

```java
@Slf4j
public class SeekConsumer extends BaseConsumer{

    @Override
    public void consume() {
        Properties props = new Properties();
        // 手动提交offset
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");

        KafkaConsumer<String, String> consumer = buildConsumer(props);

        // 消费所有消息
        List<TopicPartition> topicPartitions = Arrays.asList(new TopicPartition(TOPIC_NAME, 0), new TopicPartition(TOPIC_NAME, 1));
        consumer.assign(topicPartitions);
        consumer.seekToBeginning(topicPartitions);
        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
            for (ConsumerRecord<String, String> record : records) {
                log.info("receive message：partition : {}, offset : {}, content : {}", record.partition(), record.offset(), record.value());
            }

            if (records.count() > 0) {
                // 手动异步提交offset
                consumer.commitAsync((offsets, exception) -> {
                    if (exception != null) {
                        log.error("{} commit failed...", offsets);
                        log.error("commit failed exception: " + exception);
                    }
                });

            }
        }
    }
}
```

#### 拉取指定时间段的消息

```java
@Slf4j
public class DurationConsumer extends BaseConsumer{

    @Override
    public void consume() {
        Properties props = new Properties();
        // 手动提交offset
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
        KafkaConsumer<String, String> consumer = buildConsumer(props);

        // 1. 根据topic name查找所有的partition，并指定给consumer
        // 2. 维护一个partition和消费时间戳(比如拉去一个小时前的消息，就是当前时间戳-1个小时的时间戳)的map
        List<PartitionInfo> partitionInfos = consumer.partitionsFor(TOPIC_NAME);
        long consumeTime = new Date().getTime() - 1000 * 60 * 60;
        Map<TopicPartition, Long> map = new HashMap<>(16);
        List<TopicPartition> partitions = new LinkedList<>();

        for (PartitionInfo partitionInfo : partitionInfos) {
            TopicPartition partition = new TopicPartition(TOPIC_NAME, partitionInfo.partition());
            partitions.add(partition);
            map.put(new TopicPartition(TOPIC_NAME, partitionInfo.partition()), consumeTime);
        }
        consumer.assign(partitions);

        // 2.按照时间戳查找指定分区的offset，会返回每个分区时间戳大于或等于给定时间戳的最早的那个offset
        Map<TopicPartition, OffsetAndTimestamp> partitionAndOffsetTimeMap = consumer.offsetsForTimes(map);

        // 3.设置offset
        for (Map.Entry<TopicPartition, OffsetAndTimestamp> entry : partitionAndOffsetTimeMap.entrySet()) {
            TopicPartition partition = entry.getKey();
            OffsetAndTimestamp offsetAndTimestamp = entry.getValue();

            if (null != partition && null !=  offsetAndTimestamp){
                long offset = offsetAndTimestamp.offset();
                consumer.seek(partition, offset);
            }
        }

        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
            for (ConsumerRecord<String, String> record : records) {
                log.info("receive message：partition : {}, offset : {}, content : {}", record.partition(), record.offset(), record.value());
            }

            if (records.count() > 0) {
                consumer.commitAsync((offsets, exception) -> {
                    if (exception != null) {
                        log.error("{} commit failed...", offsets);
                        log.error("commit failed exception: " + exception);
                    }
                });

            }
        }
    }
}
```







