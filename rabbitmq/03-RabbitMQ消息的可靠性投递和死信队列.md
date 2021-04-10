# 消息的可靠性投递和死信队列

## 消息投递的过程

前面说了RabbitMq的几种工作模式，到此我们也可以大概总结书RabbitMq投递一个消息到消费端所经历的过程了，如果用一张图来表示，大概可以分为下面几步：

1. Step1：生产者发布消息到交换机
2. Step2：交换机根据路由key将消息投递到指定的队列
3. Step3：消费者监听队列，从队列种拉取消息进行消费，消息被拉取后，就会被队列删除

这其中每一步都是不可靠的，都有可能造成消息的丢失。所以每一步RabbitMq也都为我们提供了一些保证措施。



![](/img/消息可靠性投递.png)



## Step1：消息确认

生产者发布消息到交换机这个过程，主要的责任人在交换机，作为消息的接收者，它有义务告诉消息的生产者是否成功收到了消息，如果没有收到应该反馈给生产者，生产者可以考虑做一些补偿性的措施，比如记录日志，重发消息等。这个过程就叫做消息的**确认机制**，这个是在AMQP 0.9.1 protocol提供的扩展功能，所以需要`confirmSelect()`这个方法开启确认模式。他一共有三种确认机制，我们一个一个看：

### 逐个消息确认（Publishing Messages Individually）

这个是最简单的方式，也就是我消息生产者每发一次消息，就同步等待MQ确认，如果返回false或者超时时间内没有返回，说明发送失败，消息生产者可以选择重发或者其他补救措施。这种方式相对简单，但是有一个缺点，就是**同步阻塞**的，在等待确认的时候，不允许发其他消息，效率不高。

```java
/**
  * 每发一条消息就同步确认
  */
private static void publishMessagesIndividually() throws Exception {
    Connection connection = ConnectionUtil.getConnection();
    Channel channel = connection.createChannel();
    // 开启消息确认机制
    channel.confirmSelect();

    long start = System.nanoTime();
    for (int i = 0; i < MESSAGE_COUNT; i++) {
        channel.basicPublish("", TASK_QUEUE_NAME, null, BYTE_MESSAGE);
		// boolean waitForConfirms(5_000); 这个方法是返回boolean值，表示是否确认，下面的方法通过抛出异常表示确认
        channel.waitForConfirmsOrDie(5_000);
    }
    long end = System.nanoTime();
    System.out.format("Published %,d messages individually in %,d ms%n", MESSAGE_COUNT, Duration.ofNanos(end - start).toMillis());
}
```

### 批量确认（Publishing Messages in Batches）

为了优化上面的问题，我们将单个确认的方式改成批量的，我们可以选择同时发布一批消息，等待这一批消息被确认。这里我们一共发布1000条消息，每100条为一个批次，进行一次确认。这种虽然在一定程度上提高了吞吐量，但是呢还有两个问题，其一是他**仍然是同步阻塞的**，其二是如果发生了故障，我们不知道具体哪条消息发生了故障，因此可能要将这一批消息都重发一次，就会带来**重复消费**的问题。

```java
/**
  * 批量确认，这里我们一共发布1000条消息，每次发完100条，我们进行一次确认
  */
private static void publishMessagesInBatch() throws Exception {
    Connection connection = ConnectionUtil.getConnection();
    Channel channel = connection.createChannel();
    channel.confirmSelect();

    int batchSize = 100;
    int outstandingMessageCount = 0;

    long start = System.nanoTime();
    for (int i = 0; i < MESSAGE_COUNT; i++) {
        channel.basicPublish("", TASK_QUEUE_NAME, null, BYTE_MESSAGE);
        outstandingMessageCount++;
		
        // 没发一次计数器加1，到了100后确认这一批消息，计数器置0
        if (outstandingMessageCount == batchSize) {
            channel.waitForConfirmsOrDie(5_000);
            outstandingMessageCount = 0;
        }
    }
    long end = System.nanoTime();
    if (outstandingMessageCount > 0) {
        channel.waitForConfirmsOrDie(5_000);
    }
    System.out.format("Published %,d messages in batch in %,d ms%n", MESSAGE_COUNT, Duration.ofNanos(end - start).toMillis());
}
```

### 异步确认（ Handling Publisher Confirms Asynchronously）

最后呢，为了彻底解决同步的问题，RabbitMQ也为我们提供了异步确认的方式，这样我们就可以在消息生产者的客户端注册两个回调函数，一个在消息被确认的时候触发，一个在消息被拒收的时候触发。每个回调函数有两个参数，一个是消息的**唯一ID**，一个是boolean值，表示是否批量确认的。下面我们看一下代码。

```java
private static void handlePublishConfirmsAsynchronously() throws Exception {
    Connection connection = ConnectionUtil.getConnection();
    Channel channel = connection.createChannel();
    channel.confirmSelect();

    // 用一个hash表保存每一条发出的消息，key: 消息的唯一ID  value: 消息内容
    ConcurrentNavigableMap<Long, String> outstandingConfirms = new ConcurrentSkipListMap<>();

    // 消息被确认的回调函数
    ConfirmCallback ackCallback = (sequenceNumber, multiple) -> {
        // 如果批量确认，因为我们用了一个有序map，所以可以拿到对应范围内的消息ID，然后将他们从map中移除
        if (multiple) {
            ConcurrentNavigableMap<Long, String> confirmed = outstandingConfirms.headMap(
                sequenceNumber, true
            );
            confirmed.clear();
        } else {
            // 只确认了一条消息的话，就一个一个移除
            outstandingConfirms.remove(sequenceNumber);
        }
    };

    // 消息被拒收的时候触发的回调函数，打印消息内容和消息的序列号
    ConfirmCallback nackCallback = (sequenceNumber, multiple) -> {
        String body = outstandingConfirms.get(sequenceNumber);
        System.err.format(
            "Message with body %s has been nack-ed. Sequence number: %d, multiple: %b%n",
            body, sequenceNumber, multiple
        );
        ackCallback.handle(sequenceNumber, multiple);
    };

    // 将上面两个回调函数注册到channel
    channel.addConfirmListener(ackCallback, nackCallback);

    long start = System.nanoTime();
    for (int i = 0; i < MESSAGE_COUNT; i++) {
        // 发出的每一条消息都保存在一个map中
        outstandingConfirms.put(channel.getNextPublishSeqNo(), "Hello World" + i);
        channel.basicPublish("", TASK_QUEUE_NAME, null, BYTE_MESSAGE);
    }

    // 发完消息，等60s之内如果还没有确认完成，就结束程序
    if (!waitUntil(Duration.ofSeconds(60), outstandingConfirms::isEmpty)) {
        throw new IllegalStateException("All messages could not be confirmed in 60 seconds");
    }

    long end = System.nanoTime();
    System.out.format("Published %,d messages and handled confirms asynchronously in %,d ms%n", MESSAGE_COUNT, Duration.ofNanos(end - start).toMillis());
}
```

同时发送1000条消息，并开启确认机制，多次运行上述三种方式，测出性能对比如下。第一种方式是最慢的，第二种方式最快，第三种虽然介于中间，但是比第一种强很多，并且它可以获取到确认失败的消息的详细情况。

```
Published 1,000 messages individually in 347 ms
Published 1,000 messages in batch in 120 ms
Published 1,000 messages and handled confirms asynchronously in 132 ms
```

## Step2：消息回退

当消息被成功发送到交换机后，交换机就会将消息发送到队列，那么这个过程如果失败，消息则会被回退到生产者这边，那么生产者这边会注册一个监听器，监听到如果有消息被退回，会根据情况做一些补偿性措施。这个叫做消息的**回退机制**。回退机制和上述的确认机制都是在生产者这边来保证消息的可靠性的。

```java
/**
 * @author : HXY
 * @date : 2021-04-07 21:58
 **/
public class MessageReturn {

    public static void main(String[] args) throws Exception {
        testReturn();
    }

    private static void testReturn() throws Exception {
        Connection connection = ConnectionUtil.getConnection();
        Channel channel = connection.createChannel();
        channel.confirmSelect();

        // 监听回退机制
        channel.addReturnListener(returnMessage -> {
            // returnMessage封装了消息内容，退回原因等
            System.err.println("消息未发送到队列：" + new String(returnMessage.getBody()) + " 被退回");
            System.err.println("退回原因：" + returnMessage.getReplyText());
            // 重发消息等补救措施
        });
        channel.basicPublish("", "TASK_QUEUE_NAME", true, null, "Hello World".getBytes(StandardCharsets.UTF_8));
    }
}
```

## Step3：消费者签收消息

当消息成功到达队列，被消费者消费的时候，可能有两种结果：第一种当然是正常消费，处理业务逻辑，但是如果在收到消息后代码出现了异常，那么这个时候就会发生消息丢失的情况。RabbitMQ也考虑到了这个问题，因此他在消费者这边也提供了消息的确认机制。主要有两种方式：

### 自动确认

就是当消息一旦被消费者接收到，则自动确认收到，并将相应消息从队列中移除。但是在实际业务处理中，很可能接收到消息，可是业务处理出现异常，那么该消息就会丢失。

```java
public static void main(String[] args) throws IOException {
    Connection connection = ConnectionUtil.getConnection();
    Channel channel = connection.createChannel();

    channel.queueDeclare(TASK_QUEUE_NAME, true, false, false, null);
    channel.basicQos(PRE_FETCH_COUNT);

    DeliverCallback deliverCallback = (consumerTag, delivery) -> {
        String message = new String(delivery.getBody(), StandardCharsets.UTF_8);
        System.out.println("收到消息：" + message);
        // 手动签收消息
        channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
    };
    channel.basicConsume(TASK_QUEUE_NAME, false, deliverCallback, consumerTag -> {});
}
```

### 手动确认

如果设置了手动确认，就是在业务处理成功后，通过调用`channel.basicAck()`，手动确认，如果业务处理过程中发生了异常，可以在异常处理中调用`channel.basicNack()`方法拒绝签收消息，让其自动重新发送消息。下面这段代码让被拒签的消息重新放回队列进行消费，这样的问题就是有可能一条消息不断的被放回队列重新消费，总也成功不了，这当然不是个好办法。因此也可以选择抛弃这条消息，但是如果消息不重要抛弃也就抛弃了，如果这条消息很重要呢，这个时候就可以用死信队列去解决，见下文。

```java
public static void main(String[] args) throws IOException {
    Connection connection = ConnectionUtil.getConnection();
    Channel channel = connection.createChannel();

    channel.queueDeclare(TASK_QUEUE_NAME, true, false, false, null);
    channel.basicQos(PRE_FETCH_COUNT);

    DeliverCallback deliverCallback = (consumerTag, delivery) -> {
        try {
            int num = 1 / 0;
            String message = new String(delivery.getBody(), StandardCharsets.UTF_8);
            System.out.println("收到消息：" + message);
            channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
        } catch (Exception e) {
            // param1: 消息的唯一ID
            // param2: 是否批量消息
            // param3: true: 返回到队列重新消费，false: 丢弃或放到死信队列
            channel.basicNack(delivery.getEnvelope().getDeliveryTag(), false, true);
            e.printStackTrace();
        }
    };
    channel.basicConsume(TASK_QUEUE_NAME, false, deliverCallback, consumerTag -> {});
}
```

## 死信队列

死信队列，固然就是存放死信的队列，那么什么样的消息是死信呢？上面我们就提到了一种，那就是被消费者拒签的消息。当然这只是一种，在RabbitMQ中，一共有三种消息可以被定义为死信。

1. 被消费者拒签的消息
2. 原队列中的消息由于设置了TTL过期时间过期的消息
3. 队列消息长度达到限制

### 被消费者拒签的消息

当通过手动方式拒签了消息，这个消息就变成了**死信**后，那么如果这条**死信**不重要，可以选择被丢弃，但是如果很重要呢，他又该何去何从？RabbitMQ为我们提供了**死信队列**。可以让这些被拒签的**死信**转移到死信队列被保存起来，防止消息的丢失。当消息到达死信队列后，我们可以根据实际需求去做对应的处理。

![](img\死信队列-1.png)

创建一个死信交换机和死信队列，进行绑定，对应上图红色部分。

![](\img\死信队列和死信交换机的绑定.png)

将队列和上图所创建的死信交换机进行绑定。

![](\img\普通队列和死信交换机的绑定.png)

```java
public static void main(String[] args) throws IOException {
    Connection connection = ConnectionUtil.getConnection();
    Channel channel = connection.createChannel();

    // 正常队列的绑定，扩展属性选上绑定的死信交换机
    Map<String, Object> props = new HashMap<>();
    props.put("x-dead-letter-exchange", DEAD_EXCHANGE);
    channel.queueDeclare(TASK_QUEUE_NAME, true, false, false, props);

    // 声明死信队列并和死信交换机绑定，对应上图红色部分的实现
    channel.queueDeclare(DEAD_QUEUE, true, false, false, null);
    channel.queueBind(DEAD_QUEUE, DEAD_EXCHANGE, "");

    DeliverCallback deliverCallback = (consumerTag, delivery) -> {
        try {
            int num = 1 / 0;
            String message = new String(delivery.getBody(), StandardCharsets.UTF_8);
            System.out.println("收到消息：" + message);
            channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
        } catch (Exception e) {
            channel.basicNack(delivery.getEnvelope().getDeliveryTag(), false, false);
            e.printStackTrace();
        }
    };
    channel.basicConsume(TASK_QUEUE_NAME, false, deliverCallback, consumerTag -> {});
}
```

然后呢，我们写一个demo，模拟消费者正常消费队列中的消息，但是由于代码问题抛出了异常，拒绝了消息，消息被丢尽死信队列。

消费前：

![](\img\消费前.png)

执行上述代码后，消息进入死信队列

![](\img\消费后.png)

### 原队列中的消息由于设置了TTL过期时间被过期淘汰的消息

#### TTL

RabbitMQ是允许为队列或者队列中的消息设置一个过期时间的。具体参考官网文档：[Queue and Message TTL](https://www.rabbitmq.com/ttl.html)

1. 通过`x-expires`可以为一个队列设置过期时间，单位是毫秒，当过了设置的时间，队列就会被删除。
2. 通过`x-message-ttl`可以为投递到队列中的消息设置一个过期时间，单位也是毫秒，当过了这个指定的时间，消息就会过期。那么如果我们为这个队列绑定了一个死信队列，这个过期的消息就会被投递到死信队列中。通过这个机制我们可以模式一个延迟队列。

#### 延迟队列

如图所示，我们为黄色的队列绑定了一个死信队列，也设置了队列中的消息过期时间为10秒。那么当消息在到达黄色队列中的时间超过了10秒后就会被转移到死信队列中，然后我们的消费者从死信队列中消费消息，就相当于拿到的是一个延时10秒的消息。

![](\img\延迟队列.png)

这个延迟队列在什么场景有用呢，可能常见的就是我们在美团或者12306买票的时候，如果下了订单但是没有支付，那么这个订单就会在半小时后会被取消。这个我们就可以通过上面说的这种方案实现。

### 队列消息长度达到限制

RabbitMq中的队列在创建的时候可以通过`x-max-length`指定队列的长度。那么假设我们指定了`x-max-length = 3`。那么如果这个队列中已经有了三条消息，当你在往里面投递第四条消息的时候，就会投递失败。这第四条消息也就会成为一个**死信**被投递到绑定的**死信队列**中。