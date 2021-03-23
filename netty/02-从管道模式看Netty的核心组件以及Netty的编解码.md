# 从管道模式看Netty的核心组件以及Netty的编解码

## 什么是编解码

关于编解码，这里先和大家看一下例子，下面这段代码实现了一个很简单的功能，启动服务端，在启动客户端，当客户端启动成功建立连接后，会向服务端发送一个字符串`Hello World`，服务端收到数据后打印这条消息。但是如果我们把**flag1**和**flag2**处的代码注释掉，会发现服务端是收不到这条消息的。当我们把上述两行代码放开，服务端才能收到消息。这是为什么呢？因为网络只能传输二进制的流数据，是不能传递字符串对象这些东西的。因此我们必须在将数据发送到服务端之前将其转化成二进制，这个过程就叫做**编码**，而服务端收到二进制的数据，也不能直接用，需要反序列化成原本的字符串或者对象，这个过程就叫**解码**。**flag1**和**flag2**标记处的代码就是Netty为我们提供的编解码器。

```java
public class NettyServer {

    public static void main(String[] args) {
        EventLoopGroup boss = new NioEventLoopGroup();
        EventLoopGroup worker = new NioEventLoopGroup();

        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(boss, worker)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        ChannelPipeline pipeline = ch.pipeline();
                        // falg1
                        pipeline.addLast(new StringDecoder());
                        pipeline.addLast(new NettyServerHandler());
                    }
                });

        System.out.println("netty server start...");
        ChannelFuture future;
        try {
            future = bootstrap.bind(8080).sync();
            future.channel().closeFuture().sync();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            boss.shutdownGracefully();
            worker.shutdownGracefully();
        }
    }
}

public class NettyServerHandler extends SimpleChannelInboundHandler<String> {

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String msg) throws Exception {
        System.out.println(" ======> [server] received message: " + msg);
    }
}
```

```java
public class NettyClient {
    public static void main(String[] args) {
        EventLoopGroup group = new NioEventLoopGroup();
        Bootstrap bootstrap = new Bootstrap();
        bootstrap.group(group).channel(NioSocketChannel.class)
                .handler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        ChannelPipeline pipeline = ch.pipeline();
                        // falg2
                        pipeline.addLast(new StringEncoder());
                        pipeline.addLast(new NettyClientHandler());
                    }
                });
        System.out.println("netty client start...");

        try {
            bootstrap.connect("127.0.0.1", 8080).sync().channel();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            group.shutdownGracefully();
        }
    }
}

public class NettyClientHandler extends SimpleChannelInboundHandler<String> {

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String msg) throws Exception {

    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        ctx.writeAndFlush("hello world");
    }
}
```

## 责任链模式

这里我们先复习一下责任链模式和管道模式，因为下面会用到。那么什么是责任链模式呢，我们还是通过一段代码来看。假设我现在要实现一个需求就是打印日志，但是我需要根据日志级别判断是打印ERROR级别还是INFO级别。可能最简单的方式就是下面这种方式：

```java
public class PrintLogger {

    public static final int INFO = 1;
    public static final int DEBUG = 2;
    public static final int ERROR = 3;

    protected int level;

    public void logMessage(int level, String message){
        if(this.level = INFO) {
            // 打印INFO日志
        } else if (this.level = DEBUG) {
            // 打印DEBUG日志
        } else {
            // 打印ERROR日志
        }
    }
}
```

这样写虽然容易理解，但是有两个严重的问题

1. 代码耦合度高，如果我们将来又要加一个TRANCE级别的日志，那么就得在这里在加一个if条件，而且要想改动打印INFO级别的日志的逻辑，也得改这个代码
2. 不好维护，如果我们的业务越来越复杂，这里的if条件越来越多，慢慢的这个代码也会越来越长，越难维护。因此最好能将每个if条件下的逻辑解耦出来。这里就用到了前辈们给我们总结的责任链模式。

所谓责任链模式结合上述示例，就是类似于维护一个链表，如果有请求来，那么我第一个节点先看能否处理请求，如果能就直接处理返回，如果不能，这个节点则会将请求转交给下一个责任节点去处理。这样未来如果有新的逻辑，我们就加到这个链表中去就行。说起来可能有些抽象，不如直接上代码。

```java
// 抽象的处理器
public abstract class AbstractLogger {

    public static final int INFO = 1;
    public static final int DEBUG = 2;
    public static final int ERROR = 3;

    protected int level;

    // 责任链中的下一个元素
    protected AbstractLogger nextLogger;

    public void setNextLogger(AbstractLogger nextLogger){
        this.nextLogger = nextLogger;
    }

    // 核心逻辑，负责接受请求，如果能处理就处理，否则交给责任链中的下一个节点处理
    public void logMessage(int level, String message){
        if(this.level <= level){
            write(message);
        }
        if(nextLogger !=null){
            nextLogger.logMessage(level, message);
        }
    }

    // 写日志的方法，供子类实现
    abstract protected void write(String message);
}
```

```java
// 不同的处理器
public class InfoLogger extends AbstractLogger{

    public InfoLogger(int level){
        this.level = level;
    }

    @Override
    protected void write(String message) {
        System.out.println("Info ::Logger: " + message);
    }
}

public class ErrorLogger extends AbstractLogger{

    public ErrorLogger(int level){
        this.level = level;
    }

    @Override
    protected void write(String message) {
        System.out.println("Error ::Logger: " + message);
    }
}

public class DebugLogger extends AbstractLogger{

    public DebugLogger(int level) {
        this.level = level;
    }

    @Override
    protected void write(String message) {
        System.out.println("Debug ::Logger: " + message);
    }
}
```

```java
public class Main {
    // 构建一个责任链
    private static AbstractLogger getChainOfLoggers(){

        AbstractLogger errorLogger = new ErrorLogger(AbstractLogger.ERROR);
        AbstractLogger fileLogger = new DebugLogger(AbstractLogger.DEBUG);
        AbstractLogger consoleLogger = new InfoLogger(AbstractLogger.INFO);

        errorLogger.setNextLogger(fileLogger);
        fileLogger.setNextLogger(consoleLogger);

        return errorLogger;
    }

    public static void main(String[] args) {
        AbstractLogger loggerChain = getChainOfLoggers();
        loggerChain.logMessage(AbstractLogger.INFO, "This is a info message..");
    }
}
```

## pipeLine（管道）模式

那责任链模式呢，他核心是一个责任的传递，找到能处理这个请求的责任节点处理完就结束了，那有的场景可能就不合适了，比如我们下单买东西可能经过这么几个步骤：检查库存、计算金额、下单、提交物流、发送短信、发送邮件等，这些步骤每一步都是必经的，并且有一个严格的顺序。这个时候也许你可能会写出这样的代码。

```java
void buy() {
    checkStock();
    caculateAmount();
    ...
}
```

这样也许很容易理解。但是不易扩展（比如日后如果这个流程业务变化还要改核心代码），而且不好实现代码重用（也许检查库存，计算金额这些代码其他模块也能用到呢）。那么如果这个时候我们将上面的责任链模式进行一些变化，我们将检查库存、计算金额这些业务都抽象成一个个单独的类，然后加到责任链（管道）中，让他们按顺序经过每一个节点完成对应的处理（一个节点处理完成转给下一个节点不结束程序）。然后在加一个上下文的Context维护每一个节点都用到的共享数据。岂不是既解决了问题，而且解决了上面说到的不好扩展和不好实现代码重用（将每个业务抽象出来成单独的类，谁都可以用）的问题。

也许讲了这么一大堆还是有些糊涂，我们画一张图解释一下管道模式吧。这次我们就说netty处理网络请求的例子。我们知道一个请求从客户端发出，要经过编码-》服务端接受到进行解码-》服务端进行业务逻辑的处理-》在编码将消息发给客户端这么多流程。其中编码、解码等逻辑是处理每一个网络请求都能复用的。因此我们将这些都抽象成一个个handler，放在一个有方向的管道中，让请求经过这个管道中的每一个节点去处理。其中context存储上下文数据。

![](img\管道模式.png)

下面我们给出代码实现，方便进一步理解：

```java
/**
 * 上下文容器
 * 
 * @author : HXY
 * @date : 2021-03-04 21:07
 **/
@Data
public class BaseContext {

    private String request;

    public BaseContext(String request) {
        this.request = request;
    }
}
```

```java
/**
 * Handler抽象接口，提供一个deal方法让实现类实现
 * 
 * @param <C>
 */
public interface Handler<C extends BaseContext> {

    void deal(C context);
}
```

```java
// 三个具体的handler，都实现了Handler接口

public class EncodeHandler implements Handler<BaseContext>{

    @Override
    public void deal(BaseContext context) {
        System.out.println("encode..." + context.getRequest());
    }
}

public class DecodeHandler implements Handler<BaseContext>{

    @Override
    public void deal(BaseContext context) {
        System.out.println("decode..." + context.getRequest());
    }
}

public class ComputeHandler implements Handler<BaseContext> {

    @Override
    public void deal(BaseContext context) {
        System.out.println("deal..." + context.getRequest());
    }
}
```

```java
/**
 * 管道
 * 
 * @author : HXY
 * @date : 2021-03-04 21:20
 **/
@Data
public class ChainPipeLine {

    // 上下文
    private BaseContext context;

    // 借用一个list来存储handler
    private List<Handler<BaseContext>> pipeLine = new ArrayList<>();

    public ChainPipeLine(BaseContext context) {
        this.context = context;
    }

    // 核心逻辑，遍历所有handler，让数据流转到每一个handler
    public void doHandler() {
        for (Handler<BaseContext> handler : pipeLine) {
            handler.deal(context);
        }
    }

    // 提供向管道中动态添加handler的方法
    public void addHandler(Handler<BaseContext> handler) {
        pipeLine.add(handler);
    }
}
```

```java
public class Main {

    public static void main(String[] args) {
        BaseContext context = new BaseContext("message");
        ChainPipeLine pipeLine = new ChainPipeLine(context);

        pipeLine.addHandler(new EncodeHandler());
        pipeLine.addHandler(new ComputeHandler());
        pipeLine.addHandler(new DecodeHandler());

        pipeLine.doHandler();
    }
}

// 执行结果
encode...message
deal...message
decode...message
```

## Netty对管道模式的实践

了解了管道模式，那么我们也就了解了最开始看到的Netty两个核心组件：`ChannelPipeline`和`ChannelHandler`。`ChannelPipeline`就是管道模式中的管道，其实底层是一个有头节点有尾节点的**双向链表**。而`ChannelHandler`就是管道模式中的Handler，它作为一个顶级的接口，提供了很多方法供其他的handler实现，那么我们编解码的`StringDecoder`和`StringEncoder`就实现了编解码的功能。`ChannelHandlerContext`就是管道模式中那个上下文了。

![](img\ChannelPipeLine.png)

这里我们可能着重讲的是`ChannelHandler`，他又有两个核心的子接口，`ChannelOutboundHandler`和`ChannelOutboundHandler`。从命名提炼出来核心的区别就是`Out`和`In`。究竟有什么用呢。我们这样想 ，请求从客户端发出，到服务端，服务端响应给客户端，对客户端或者服务端来说，都有两个动作，**出**和**入**。那么针对出和入这两个不同的动作，有些处理总是有区别的。比如**编码**就应该在消息**发出**这个过程进行，而**解码**就应该在消息**收入**这个过程进行。那么一个管道中既有编码又有解码的Handler，那么如何防止在消息发出的时候错走了解码，消息接受的时候错走了编码呢？这个时候`ChannelOutboundHandler`和`ChannelOutboundHandler`就有用武之地了。凡是实现了`ChannelOutboundHandler`接口的Handler，我们就知道他是负责处理消息**出**逻辑的，`ChannelOutboundHandler`反之。

![](img\ChannelHandler.png)





## Netty提供的编解码器

Netty为我们提供了很多编解码器，比如编解码字符串的`StringDecoder`和`StringEncoder`和编解码对象的`ObjectDecoder`和`ObjectEncoder`。我们通过字符串编解码器看他是如何实现编解码的。

首先看`StringDecoder`，作为一个解码器，他是在**消息入**的时候用的，所以肯定是实现了`ChannelInboundHandler`的。下文是他的部分源码，其中继承的`MessageToMessageDecoder`实现了`ChannelInboundHandler`。

```java
@Sharable
public class StringDecoder extends MessageToMessageDecoder<ByteBuf> {

    // 解码的核心逻辑，其实就是把二进制的字节转换成字符串。这个方法什么时候被调用的呢？看父类
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf msg, List<Object> out) throws Exception {
        out.add(msg.toString(charset));
    }
}
```

```java
// channelRead，当通道有读取事件的时候会触发
// 父类定义如果有数据读取到，就调用解码方法，而解码的逻辑又交给不同的子类实现，又是对模板方法设计模式的应用
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    CodecOutputList out = CodecOutputList.newInstance();
    try {
        if (acceptInboundMessage(msg)) {
            @SuppressWarnings("unchecked")
            I cast = (I) msg;
            try {
                // 这里调用了解码的方法
                decode(ctx, cast, out);
            } finally {
                ReferenceCountUtil.release(cast);
            }
        } else {
            out.add(msg);
        }
    } catch (DecoderException e) {
        throw e;
    } catch (Exception e) {
        throw new DecoderException(e);
    } finally {
        try {
            int size = out.size();
            for (int i = 0; i < size; i++) {
                ctx.fireChannelRead(out.getUnsafe(i));
            }
        } finally {
            out.recycle();
        }
    }
}

protected abstract void decode(ChannelHandlerContext ctx, I msg, List<Object> out) throws Exception;
```

`StringEncoder`逻辑和上面大概雷同，不在赘述。

## 自定义编解码器

了解了基本套路后，如何实现一个自定义编解码，其实大体的模板代码只是照猫画虎的事情，核心在于选用一个高效给力的序列化工具来进行编解码。这里推荐**protostuff**，据说性能很不错，比java提供的序列化工具要优秀很多。

**protostuff**使用案例

1. 引入依赖

```xml
<dependency>
    <groupId>com.dyuproject.protostuff</groupId>
    <artifactId>protostuff-api</artifactId>
    <version>1.0.10</version>
</dependency>
<dependency>
    <groupId>com.dyuproject.protostuff</groupId>
    <artifactId>protostuff-core</artifactId>
    <version>1.0.10</version>
</dependency>
<dependency>
    <groupId>com.dyuproject.protostuff</groupId>
    <artifactId>protostuff-runtime</artifactId>
    <version>1.0.10</version>
</dependency>
```

2. 实现序列化反序列化的方法

```java
public class ProtostuffUtil {
    private static Map<Class<?>, Schema<?>> cachedSchema = new ConcurrentHashMap<>();

    private static <T> Schema<T> getSchema(Class<T> clazz) {
        @SuppressWarnings("unchecked")
        Schema<T> schema = (Schema<T>) cachedSchema.get(clazz);
        if (schema == null) {
            schema = RuntimeSchema.getSchema(clazz);
            if (schema != null) {
                cachedSchema.put(clazz, schema);
            }
        }
        return schema;
    }

    /**
     * 序列化
     *
     * @param obj
     * @return
     */
    public static <T> byte[] serializer(T obj) {
        @SuppressWarnings("unchecked")
        Class<T> clazz = (Class<T>) obj.getClass();
        LinkedBuffer buffer = LinkedBuffer.allocate(LinkedBuffer.DEFAULT_BUFFER_SIZE);
        try {
            Schema<T> schema = getSchema(clazz);
            return ProtostuffIOUtil.toByteArray(obj, schema, buffer);
        } catch (Exception e) {
            throw new IllegalStateException(e.getMessage(), e);
        } finally {
            buffer.clear();
        }
    }

    /**
     * 反序列化
     *
     * @param data
     * @param clazz
     * @return
     */
    public static <T> T deserializer(byte[] data, Class<T> clazz) {
        try {
            T obj = clazz.newInstance();
            Schema<T> schema = getSchema(clazz);
            ProtostuffIOUtil.mergeFrom(data, obj, schema);
            return obj;
        } catch (Exception e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
    }

    public static void main(String[] args) {
        byte[] userBytes = ProtostuffUtil.serializer(new User(1, "hanxingyu"));
        User user = ProtostuffUtil.deserializer(userBytes, User.class);
        System.out.println(user);
    }
}
```

3. 实现自定义的编解码器的Handler，在其中的decode和encode方法中使用上述工具类。