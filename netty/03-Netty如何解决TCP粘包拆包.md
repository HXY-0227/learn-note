# Netty中的粘包拆包

## 粘包拆包问题重现

关于什么是粘包拆包呢，我们先通过一个demo来看一下。

Server端代码

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
                        pipeline.addLast(new StringDecoder());
                        pipeline.addLast(new StringEncoder());
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

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        System.err.println("=== " + ctx.channel().remoteAddress() + " is active ===");
    }
}
```

Client端代码

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
                        pipeline.addLast(new StringEncoder());
                        pipeline.addLast(new StringDecoder());
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
        for (int i = 0; i < 100; i++) {
            ctx.writeAndFlush("hello world");
        }
    }
}
```

上面这段代码主要实现一个功能，启动服务端，然后在启动客户端，客户端启动后会循环向服务端发送100次**hello world**，然后服务端收到消息打印一下。那么我们看一下执行的结果会是什么样。执行后我们发现并没有像我们预期的一样，到一次收到消息打印一次hello world。而是每次收到的很没有规律，有时候一次收到三四个消息，有时候一连收到十几个消息，这个呢就叫**粘包**。

```
 ======> [server] received message: hello worldhello world
 ======> [server] received message: hello worldhello worldhello worldhello world
 ======> [server] received message: hello worldhello worldhello worldhello worldhello worldhello worldhello worldhello worldhello world
 ======> [server] received message: hello worldhello world
 ======> [server] received message: hello world
 // 删去一部分
```

## 粘包拆包的原因

那么为什么会造成上面这种状况呢？主要是因为TCP协议是基于流传输的协议，数据在整个传输过程中是没有界限区分的，如果一个包过大的话，可能会被切成多个包进行分批传输，如果一个包过小的话，也会放到缓冲区，等到将好几个小包整合成一个大包进行传输。因此我们读取数据，往往不一定能获取到一个完整的预期的数据包。

## 粘包拆包的描述

上面我们浮现了一个粘包的现象，那么我们知道除了粘包，还有拆包。这里呢，我们通过几张图详细的解释一下粘包拆包。

1. 这张图中客户端向服务端发送**两个**数据包：**Hello** 和 **World**。服务端按照预期收到**两个**数据包**Hello** 和 **World**。属于正常现象。

![](img\粘包拆包-正常.png)

2. 这张图中客户端向服务端发送**两个**数据包：**Hello** 和 **World**。然而服务端只收到**一个**数据包**HelloWorld**。两个本来要分开的包粘在了一起。就叫粘包。

![](/img/粘包拆包-粘包.png)

3. 这张图中又分为出现两种情况。虽然两种情况服务端收到的数据不一样，但都有一个特点，那就是一个完整的包被拆成两部分，我们就叫做**拆包**。
   1. 第一张图中客户端向服务端发送**一个**数据包：**HelloWorld**，然而服务端收到**两个**包：**HelloWo**和**rld**。
   2. 第二张图中客户端向服务端发送**一个**数据包：**HelloWorld**，然而服务端收到**两个**包：**Hel**和**loWorld**。

![](img\粘包拆包-拆包.png)

## 粘包拆包的解决方案

Netty为了解决粘包拆包的问题，给我提供了很多方案，我们简单介绍常用的几种：

1. 根据固定长度来分包

   上面的示例代码中我们常常收到多个数据包粘到一起的情况，那么如果我们在服务端限定每次收到的消息长度固定为11，如果收到超过11位，也只截取11位作为一个包，那么其实就可以解决粘包的问题。这种方案Netty已经为我们提供了一个Handler，就是`FixedLengthFrameDecoder`，它支持传入一个int类型的参数`frameLength`，如果出现粘包，那么我们根据`frameLength`对数据包进行拆分；如果出现拆包，则可以等待下一个数据包过来，然后根据`frameLength`对下一个包进行截取和上一个不完整的包进行拼接，直到得到一个完整的包。

   然后我们对上述示例代码进行改造，就可以解决粘包拆包的问题。

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
                        // 通过FixedLengthFrameDecoder解决粘包拆包问题
                        pipeline.addLast(new FixedLengthFrameDecoder(11));
                        pipeline.addLast(new StringDecoder());
                        pipeline.addLast(new StringEncoder());
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
   ```

   

2. 特殊分隔符来分包

   除过上述的方案，我们还可以根据某一个特殊符号来进行分包，比如我们每次发送消息的时候在结尾多加一个自定义的特殊符号，那么服务端在收到数据包后，如果粘包，就根据自定义的特殊符号进行拆分；如果出现拆包，就等下一个数据包发过来拼接到特殊符号为止凑成一个完整的包，也可以解决粘包拆包。这种方案，针对这种方案，Netty为我们提供了`DelimiterBasedFrameDecoder`这个handler，他的最基础的构造函数如下：

   ```java
   // maxFrameLength 最大的长度限制，超过这个长度，会抛异常，防止数据过大，导致内存溢出
   // delimiter就是我们那个自定义的特殊符号
   public DelimiterBasedFrameDecoder(int maxFrameLength, ByteBuf delimiter) {
       this(maxFrameLength, true, delimiter);
   }
   ```

   同样对上述示例代码进行改造，也可以解决粘包拆包的问题

   ```java
   public class NettyServer {
   
       private static final String DELIMITER = "-";
   
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
                           // DelimiterBasedFrameDecoder解决粘包拆包
                           pipeline.addLast(new DelimiterBasedFrameDecoder(1024, Unpooled.wrappedBuffer(DELIMITER.getBytes())));
                           pipeline.addLast(new StringDecoder());
                           pipeline.addLast(new StringEncoder());
                           // 自定义收到心跳包如何处理的handler
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
               worker.shutdo
                   wnGracefully();
           }
       }
   }
   ```


3. 自定义分包解码器解决粘包拆包

   上面的方式总是有局限性的，比如我们按照`-`这个特殊字符分包，那么假如我们本身发的消息内容就包括`-`，就会导致一个完整的包也会被拆分。所以我们还可以自定义编解码器去解决分包拆包。这里我们提供一种方案就是每次客户端发送消息的时候都把消息本身的内容和消息的长度都发送过去，这样在服务端解码的时候就可以根据长度来解析数据包。

   ```java
   @Data
   public class MessageProtocol {
   
       // 消息长度
       private int length;
   
       // 消息内容
       private byte[] content;
   }
   ```

   ```java
   public class MessageEncoder extends MessageToByteEncoder<MessageProtocol> {
   
       @Override
       protected void encode(ChannelHandlerContext ctx, MessageProtocol msg, ByteBuf out) throws Exception {
           out.writeInt(msg.getLength());
           out.writeBytes(msg.getContent());
       }
   }
   ```

   ```java
   public class MessageDecoder extends ByteToMessageDecoder {
   
       int length = 0;
   
       @Override
       protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
           if(in.readableBytes() >= 4) {
               // 先解析出消息的长度
               if (length == 0){
                   length = in.readInt();
               }
               // 在读取收到的内容，如果收到的内容长度小于length，说明发生了拆包，在等新的包发来
               if (in.readableBytes() < length) {
                   return;
               }
               // 走到这里说明完整的包已经收到了，按照根据长度读出内容
               byte[] content = new byte[length];
               if (in.readableBytes() >= length){
                   in.readBytes(content);
   
                   // 封装成MyMessageProtocol对象，传递到下一个handler业务处理
                   MessageProtocol messageProtocol = new MessageProtocol();
                   messageProtocol.setLength(length);
                   messageProtocol.setContent(content);
                   out.add(messageProtocol);
               }
               length = 0;
           }
       }
   }
   ```

## 附用Netty实现简单的聊天室

服务端：

```java
public class ChatServer {
    public static void main(String[] args) throws InterruptedException {
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .option(ChannelOption.SO_BACKLOG, 1024)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ChannelPipeline pipeline = ch.pipeline();
                            pipeline.addLast("decoder", new StringDecoder());
                            pipeline.addLast("encoder", new StringEncoder());
                            pipeline.addLast(new ChatServerHandler());
                        }
                    });
            System.out.println("Netty server start...");
            ChannelFuture channelFuture = bootstrap.bind(9000).sync();
            // 关闭通道
            channelFuture.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}

public class ChatServerHandler extends ChannelInboundHandlerAdapter {

    /**
     * GlobalEventExecutor.INSTANCE 是全局的事件执行器，是一个单例
     */
    private static final ChannelGroup CHANNEL_GROUP = new DefaultChannelGroup(GlobalEventExecutor.INSTANCE);
    SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

    /**
     * 表示 channel 处于就绪状态, 提示上线
     */
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        Channel channel = ctx.channel();
        // 将该客户加入聊天的信息推送给其它在线的客户端
        // 该方法会将 channelGroup 中所有的 channel 遍历，并发送消息
        CHANNEL_GROUP.writeAndFlush("[ 客户端 ]" + channel.remoteAddress() + " 上线了 " + sdf.format(new
                java.util.Date())+ "\n");
        CHANNEL_GROUP.add(channel);
        ctx.fireChannelRead(ctx);
        System.out.println(ctx.channel().remoteAddress() + " 上线了"+ "\n");
    }

    /**
     * 表示 channel 处于不活动状态, 提示离线了
     */
    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        Channel channel = ctx.channel();
        // 将客户离开信息推送给当前在线的客户
        CHANNEL_GROUP.writeAndFlush("[ 客户端 ]" + channel.remoteAddress() + " 下线了"+ "\n");
        System.out.println(ctx.channel().remoteAddress() + " 下线了"+ "\n");
        System.out.println("channelGroup size=" + CHANNEL_GROUP.size());
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        // 关闭通道
        ctx.close();
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        Channel channel = ctx.channel();
        // 这时我们遍历 channelGroup, 根据不同的情况， 发送不同的消息
        CHANNEL_GROUP.forEach(ch -> {
            if (channel != ch) {
                ch.writeAndFlush("[ 客户端 ]" + channel.remoteAddress() + " 发送了消息：" + msg + "\n" + "_");
            } else {
                ch.writeAndFlush("[ 自己 ]发送了消息：" + msg + "\n");
            }
        });
    }
}
```

客户端：

```java
public class ChatClient {

    public static void main(String[] args) throws InterruptedException {
        EventLoopGroup group = new NioEventLoopGroup();
        try {
            Bootstrap bootstrap = new Bootstrap()
                    .group(group)
                    .channel(NioSocketChannel.class)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ChannelPipeline pipeline = ch.pipeline();
                            pipeline.addLast("decoder", new StringDecoder());
                            pipeline.addLast("encoder", new StringEncoder());
                            pipeline.addLast(new ChatClientHandler());
                        }
                    });
            System.out.println("netty client start");
            ChannelFuture channelFuture = bootstrap.connect("127.0.0.1", 9000).sync();
            channelFuture.channel().closeFuture().sync();
        } finally {
            group.shutdownGracefully();
        }
    }
}

public class ChatClientHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        System.out.println(msg);
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        System.out.println("========" + ctx.channel().localAddress() + "========");
        ctx.writeAndFlush("hello");
    }
}
```

## 附Netty源码路径

![](\img\Netty源码路径.png)