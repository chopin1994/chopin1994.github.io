---
title: Netty浅析
date: 2019-09-06 00:49:55
author: Chopin
tags:   #标签
    - Netty
---

##  简介

Netty是一款异步的事件驱动的网络应用程序框架，支持快速地开发可维护的高性能的面向协议的服务器和客户端。它提供了对TCP、UDP和文件传输的支持，作为一个异步NIO框架，Netty的所有IO操作都是异步非阻塞的，通过Future-Listener机制，用户可以方便的主动获取或者通过通知机制获得IO操作结果。

<!-- more -->

![](/png/netty/netty.png)

为什么用Netty不用NIO

1. NIO的类库和API繁杂，使用麻烦，你需要熟练掌握Selector、ServerSocketChannel、SocketChannel、ByteBuffer等；
2. 需要具备其它的额外技能做铺垫，例如熟悉Java多线程编程，因为NIO编程涉及到Reactor模式，你必须对多线程和网路编程非常熟悉，才能编写出高质量的NIO程序；
3. 可靠性能力补齐，工作量和难度都非常大。例如客户端面临断连重连、网络闪断、半包读写、失败缓存、网络拥塞和异常码流的处理等等，NIO编程的特点是功能开发相对容易，但是可靠性能力补齐工作量和难度都非常大；
4. JDK NIO的BUG，例如epoll bug，它会导致Selector空轮询，最终导致CPU 100%。官方声称在JDK1.6版本的update18修复了该问题，但是直到JDK1.7版本该问题仍旧存在，只不过该bug发生概率降低了一些而已，它并没有被根本解决。

##  Reactor模式和Netty线程模型

###  什么是reactor模式

Reactor模式是**事件驱动的，有一个或多个并发输入源，有一个Service Handler，有多个Request Handlers**；这个Service Handler会同步的将输入的请求多路复用的分发给相应的Request Handler。 

![](/png/netty/reactor模式.png)

###  3种reactor模式

####  单线程的reactor模式

![](/png/netty/单线程reactor模型.jpg)

Reactor线程是个多面手，负责多路分离套接字，Accept新连接，并分派请求到处理器链中。该模型适用于处理器链中业务处理组件能快速完成的场景。不过，这种单线程模型不能充分利用多核资源，所以实际使用的不多。 

性能缺陷：

1. 一个NIO线程同时处理成百上千的链路，性能上无法支撑。即便NIO线程的CPU负荷达到100%，也无法满足海量消息的编码、解码、读取和发送；
2. 当NIO线程负载过重之后，处理速度将变慢，这会导致大量客户端连接超时，超时之后往往进行重发，这更加重了NIO线程的负载，最终导致大量消息积压和处理超时，NIO线程会成为系统的性能瓶颈；
3. 可靠性问题。一旦NIO线程出现错误，或者进入死循环，会导致整个系统通讯模块不可用，不能接收和处理外部信息，造成节点故障。

####  多线程的reactor模式

![](/png/netty/多线程reactor模型.jpg)

Reactor多线程模型的特点：

1. 有专门一个NIO线程-Acceptor线程用于监听服务端，接收客户端的TCP连接请求； 
2. 网络IO操作-读、写等由一个NIO线程池负责，线程池可以采用标准的JDK线程池实现，它包含一个任务队列和N个可用的线程，由这些NIO线程负责消息的读取、解码、编码和发送； 
3. 1个NIO线程可以同时处理N条链路，但是1个链路只对应1个NIO线程，防止发生并发操作问题。

在绝大多数场景下，Reactor多线程模型都可以满足性能需求；但是，当用户进一步增加的时候，一个NIO线程负责监听和处理所有的客户端连接可能会存在性能问题。例如并发百万客户端连接，或者服务端需要对客户端握手进行安全认证，但是认证本身非常损耗性能。在这类场景下，单独一个Acceptor线程可能会存在性能不足问题。

####  主从reactor多线程模型

![](/png/netty/主从模式reactor多线程模型.jpg)

注册accepter事件处理器到mainReactor线程池中，这样mainReactor会监听客户端向服务端发起的连接请求

当客户端向服务端发起连接请求时，mainReactor监听到了该请求将事件派发给acceptor事件处理器进行处理，可通过accept方法获得连接socketChannel，然后将socketChannel传递给subReactor线程池

subReactor线程池分配一个subReactor线程给这个SocketChannel，监听I/O的read、write操作，相关业务逻辑的处理交给工作线程池来完成

###  Netty的线程模型

![](/png/netty/netty线程模型.png)

当NettyServer启动时候会创建两个NioEventLoopGroup线程池组。

boss组用来接受客户端发来的连接，在监听一个端口的情况下，一个NioEventLoop通过一个NioServerSocketChannel监听端口，处理TCP连接。worker组则负责对完成TCP三次握手的连接进行处理。

如上图每个NioEventLoopGroup里面包含了多个NioEventLoop，每个NioEventLoop中包含了一个NIO Selector、一个队列、一个线程；其中线程用来做轮询注册到Selector上的Channel的读写事件和对投递到队列里面的事件进行处理。 

##  核心组件

###  Channel接口、EventLoop接口

Channel 是 Netty 网络操作抽象类，它除了包括基本的 I/O 操作，如 bind、connect、read、write 之外，还包括了 Netty 框架相关的一些功能，如获取该 Channe l的 EventLoop。 

在传统的网络编程中，作为核心类的 Socket ，它对程序员来说并不是那么友好，直接使用其成本还是稍微高了点。而Netty 的 Channel 则提供的一系列的 API ，它大大降低了直接与 Socket 进行操作的复杂性。 

Netty 基于事件驱动模型，使用不同的事件来通知我们状态的改变或者操作状态的改变。它定义了在整个连接的生命周期里当有事件发生的时候处理的核心抽象。

Channel 为Netty 网络操作抽象类，EventLoop 主要是为Channel 处理 I/O 操作，两者配合参与 I/O 操作。

![](/png/netty/channel&eventLoop.jpg)

- 一个EventLoopGroup包含一个或者多个EventLoop
- 一个EventLoop在他的生命周期只和一个线程绑定
- 所有由EventLoop处理的I/O事件都将在它专有的线程上被处理
- 一个Channel在它的生命周期内只注册于一个EventLoop
- 一个EventLoop可能会被分配给一个或多个Channel

###  ChannelFuture接口

Netty 为异步非阻塞，即所有的 I/O 操作都为异步的，因此，我们不能立刻得知消息是否已经被处理了。Netty 提供了 ChannelFuture 接口，通过该接口的 addListener() 方法注册一个 ChannelFutureListener，当操作执行完成（成功或者失败）时，监听就会自动触发返回结果。

###  ChannelHandler、ChannelPipeline

####  ChannelHandler

ChannelHandler 为 Netty 中最核心的组件，它充当了所有处理入站和出站数据的应用程序逻辑的容器。ChannelHandler 主要用来处理各种事件，这里的事件很广泛，比如可以是连接、数据接收、异常、数据转换等。

ChannelHandler 有两个核心子类 ChannelInboundHandler 和 ChannelOutboundHandler，其中 ChannelInboundHandler 用于接收、处理入站数据和事件，而 ChannelOutboundHandler 则相反。我们经常通过一个ChannelInboundHandler的实现类来实现业务逻辑的处理。

####  ChannelPipeline

ChannelPipeline 为 ChannelHandler 链提供了一个容器并定义了用于沿着链传播入站和出站事件流的 API。一个数据或者事件可能会被多个 Handler 处理，在这个过程中，数据或者事件经流 ChannelPipeline，由 ChannelHandler 处理。在这个处理过程中，一个 ChannelHandler 接收数据后处理完成后交给下一个 ChannelHandler，或者什么都不做直接交给下一个 ChannelHandler。 

![](/png/netty/channelPipeline.jpg)

当一个数据流进入 ChannlePipeline 时，它会从 ChannelPipeline 头部开始传给第一个 ChannelInboundHandler ，当第一个处理完后再传给下一个，一直传递到管道的尾部。与之相对应的是，当数据被写出时，它会从管道的尾部开始，先经过管道尾部的 “最后” 一个ChannelOutboundHandler，当它处理完成后会传递给前一个 ChannelOutboundHandler 。

当ChannelHandler被添加到ChannelPipeline时，它将会被分配一个ChannelHandlerContext，代表了ChannelHandler和ChannelPipeline之间的绑定。

####  编码器和解码器

由于网络数据总是一系列的字节，通过Netty发送或者接受消息时，将会发生一次数据转换：入站消息会被解码，由字节转换为另一种格式，通常是一个Java对象；出站消息会被编码，从当前格式转换为字节。

Netty提供了编码器的基类MessageToByteEncoder以及解码器的基类ByteToMessageDecoder，Netty提供的所有解码器/编码器适配器类都实现了ChannelInboundHandler或者ChannelOutboundHandler接口。如果我们要自定义的编码/解码规则，只需要继承基类，实现encode()/decode()方法。

```java
	@Override
    protected void encode(ChannelHandlerContext channelHandlerContext, Object iotPacketRequest, 
                            ByteBuf out) {
        if (null == iotPacketRequest) {
            return;
        }
        String body = JsonUtils.bean2Json(iotPacketRequest);
        byte[] bodyBytes = body.getBytes(Charset.forName("utf-8"));
        out.writeShort(IotConnectProperties.MAGIC_CODE);
        out.writeShort(bodyBytes.length);
        out.writeBytes(bodyBytes);
    }
```

###  服务端启动分析

以创建一个Netty服务端为例

```java
public class NettyServer {

    public void bind(int port){
        // 创建EventLoopGroup
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            // 创建ServerBootStrap实例
            // ServerBootstrap 用于启动NIO服务端的辅助启动类，目的是降低服务端的开发复杂度
            ServerBootstrap b = new ServerBootstrap();
            // 绑定Reactor线程池
            b.group(bossGroup, workerGroup)
                    // 设置并绑定服务端Channel
                    // 指定所使用的NIO传输的Channel
                    .channel(NioServerSocketChannel.class)
                    .option(ChannelOption.SO_BACKLOG, 1024)
                    .handler(new LoggingServerHandler())
                    .childHandler(new ChannelInitializer(){
                        @Override
                        protected void initChannel(Channel ch) throws Exception {
                            ch.pipeline().addLast("decoder", new HttpRequestDecoder());
                            ch.pipeline().addLast("encoder", new HttpResponseEncoder());
                            ch.pipeline().addLast("httpServerHandler", new HttpServerHandler());
                        }
                    });

            // 绑定端口，同步等待成功
            ChannelFuture future = b.bind(port).sync();
            // 等待服务端监听端口关闭
            future.channel().closeFuture().sync();

        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            // 优雅地关闭
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```

服务端的创建主要步骤为：

1. 创建 ServerBootstrap 实例
2. 设置并绑定 Reactor 线程池
3. 设置服务端 Channel
4. 添加并设置 ChannelHandler
5. 绑定并启动监听端口

####  创建EventLoopGroup

```java
EventLoopGroup bossGroup = new NioEventLoopGroup();
EventLoopGroup workerGroup = new NioEventLoopGroup();
```

bossGroup 为 BOSS 线程组，用于服务端接受客户端的连接, workerGroup 为 worker 线程组，用于进行 SocketChannel 的网络读写。 

####  **创建ServerBootstrap实例**

```java
ServerBootstrap b = new ServerBootstrap();
```

ServerBootStrap为Netty服务端的启动引导类，用于帮助用户快速配置、启动服务端服务。 客户端的引导类是Bootstrap。ServerBootStrap 提供了如下一些方法

| 方法名称  | 方法描述                                   |
| --------- | ------------------------------------------ |
| `group`   | 设置 ServerBootstrap 要用的 EventLoopGroup |
| `channel` | 设置将要被实例化的 ServerChannel 类        |
| `option`  | 实例化的 ServerChannel 的配置项            |
| `Handler` | 设置并添加 Handler                         |
| `bind`    | 绑定 ServerChannel                         |

####  设置并绑定线程池

```java
b.group(bossGroup, workerGroup)
```

调用group()方法，为ServerBootstrap实例设置绑定reactor线程池

```java
	public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup) {
        super.group(parentGroup);        // 绑定boosGroup
        if (childGroup == null) {
            throw new NullPointerException("childGroup");
        }
        if (this.childGroup != null) {
            throw new IllegalStateException("childGroup set already");
        }
        this.childGroup = childGroup;    // 绑定workerGroup
        return this;
    }
```

####  设置服务端Channel

```
.channel(NioServerSocketChannel.class)
```

调用channel()方法设置服务端Channel类型，注意这里参数是Class对象，Netty通过工厂类，利用反射来创建NioServerSocketChannel对象

```java
	public B channel(Class<? extends C> channelClass) {
        if (channelClass == null) {
            throw new NullPointerException("channelClass");
        }
        return channelFactory(new ReflectiveChannelFactory<C>(channelClass));
    }
```

这里传递的是 ReflectiveChannelFactory，其源代码如下： 

```java
	public class ReflectiveChannelFactory<T extends Channel> implements ChannelFactory<T> {

    private final Class<? extends T> clazz;

    public ReflectiveChannelFactory(Class<? extends T> clazz) {
        if (clazz == null) {
            throw new NullPointerException("clazz");
        }
        this.clazz = clazz;
    }
    //需要创建 channel 的时候，该方法将被调用
    @Override
    public T newChannel() {
        try {
            // 反射创建对应 channel
            return clazz.newInstance();
        } catch (Throwable t) {
            throw new ChannelException("Unable to create Channel from class " + clazz, t);
        }
    }

    @Override
    public String toString() {
        return StringUtil.simpleClassName(clazz) + ".class";
    }
}

```

####  添加并设置ChannelHandler

```java
 	.handler(new LoggingServerHandler())
	.childHandler(new ChannelInitializer(){
		@Override
		protected void initChannel(Channel ch) throws Exception {
			ch.pipeline().addLast("decoder", new HttpRequestDecoder());
			ch.pipeline().addLast("encoder", new HttpResponseEncoder());
			ch.pipeline().addLast("httpServerHandler", new HttpServerHandler());
		}
	})

```

`handler()`设置的 Handler 是服务端 NioServerSocketChannel的，childHandler()`设置的 Handler 是属于每一个新建的 NioSocketChannel 的

####  绑定端口，启动服务端

绑定端口并启动服务，如下： 

```java
ChannelFuture future = b.bind(port).sync();

```

深入源码我们发现核心方法有两个`initAndRegister()`， `doBind0()`

#####  initAndRegister()

```java
final ChannelFuture initAndRegister() {
    Channel channel = null;
    // ...
    channel = channelFactory.newChannel();
    //...
    init(channel);
    //...
    ChannelFuture regFuture = config().group().register(channel);
    //...
    return regFuture;
}

```

initAndRegister做了3件事：

1.**new一个Channel**

```java
channel = channelFactory.newChannel();

```

前面在ServerBootstrap实例设置服务端Channel时，设置了这个Channel的类型，这里就通过工厂类的方法生成NioServerSocketChannel对象。

追溯NioServerSocketChannel的默认构造函数，我们可以发现在构造该实例时，设置了channel为非阻塞模式、SelectionKey.OP_ACCEPT事件、channelId 、NioMessageUnsafe(封装了用于数据传输操作的函数)、DefaultChannelPipeline和 NioServerSocketChannelConfig 属性。 

2.**init这个Channel**

```java
void init(Channel channel) throws Exception {
         // 设置配置的option参数
        final Map<ChannelOption<?>, Object> options = options0();
        synchronized (options) {
            channel.config().setOptions(options);
        }

        final Map<AttributeKey<?>, Object> attrs = attrs0();
        synchronized (attrs) {
            for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
                @SuppressWarnings("unchecked")
                AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
                channel.attr(key).set(e.getValue());
            }
        }

        // 获取绑定的pipeline
        ChannelPipeline p = channel.pipeline();

        // 准备child用到的4个part
        final EventLoopGroup currentChildGroup = childGroup;
        final ChannelHandler currentChildHandler = childHandler;
        final Entry<ChannelOption<?>, Object>[] currentChildOptions;
        final Entry<AttributeKey<?>, Object>[] currentChildAttrs;
        synchronized (childOptions) {
            currentChildOptions = childOptions.entrySet().toArray(newOptionArray(childOptions.size()));
        }
        synchronized (childAttrs) {
            currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(childAttrs.size()));
        }

        // 为NioServerSocketChannel的pipeline添加一个初始化Handler,
        // 当NioServerSocketChannel在EventLoop注册成功时，该handler的init方法将被调用
        p.addLast(new ChannelInitializer<Channel>() {
            @Override
            public void initChannel(Channel ch) throws Exception {
                final ChannelPipeline pipeline = ch.pipeline();
                ChannelHandler handler = config.handler();
                //如果用户配置过Handler
                if (handler != null) {
                    pipeline.addLast(handler);
                }

                ch.eventLoop().execute(new Runnable() {
                    @Override
                    public void run() {
                        // 为NioServerSocketChannel的pipeline添加ServerBootstrapAcceptor处理器
                        // 该Handler主要用来将新创建的NioSocketChannel注册到EventLoopGroup中
                        pipeline.addLast(new ServerBootstrapAcceptor(
                                currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                    }
                });
            }
        });
    }

```

我们发现init其实就做了三件事：

- 设置options、attrs
- 设置新接入Channel的options、attrs
- 将用于服务端注册的 Handler ServerBootstrapAcceptor 添加到 ServerChannel的ChannelPipeline 中。ServerBootstrapAcceptor 为一个接入器，专门接受新请求。

3.**向EventLoopGroup中注册这个Channel**

```java
	ChannelFuture regFuture = config().group().register(channel);

```

通过追溯我们发现过程如下：

```java
	public ChannelFuture register(Channel channel) {
        return next().register(channel);
    }

```

调用 `next()` 方法从 EventLoopGroup 中获取下一个 EventLoop，调用 `register()` 方法注册： 

```java
	public ChannelFuture register(Channel channel) {
        return register(new DefaultChannelPromise(channel, this));
    }

```

将Channel和EventLoop封装成一个DefaultChannelPromise对象，然后调用register()方法。DefaultChannelPromis为ChannelPromise的默认实现，而ChannelPromisee继承Future，具备异步执行结构，绑定Channel，所以又具备了监听的能力，故而ChannelPromis是Netty异步执行的核心接口。 

```java
	public ChannelFuture register(ChannelPromise promise) {
        ObjectUtil.checkNotNull(promise, "promise");
        promise.channel().unsafe().register(this, promise);
        return promise;
    }

```

unsafe就是我们之前构造NioServerSocketChannel时new的对象，这里调用register方法过程如下：

```java
		public final void register(EventLoop eventLoop, final ChannelPromise promise) {
            if (eventLoop == null) {
                throw new NullPointerException("eventLoop");
            }
            if (isRegistered()) {
                promise.setFailure(new IllegalStateException("registered to an event loop already"));
                return;
            }
            if (!isCompatible(eventLoop)) {
                promise.setFailure(
                        new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
                return;
            }

            AbstractChannel.this.eventLoop = eventLoop;

            // 必须要保证注册是由该EventLoop发起的
            if (eventLoop.inEventLoop()) {
                register0(promise);        // 注册
            } else {
                // 如果不是单独封装成一个task异步执行
                try {
                    eventLoop.execute(new Runnable() {
                        @Override
                        public void run() {
                            register0(promise);
                        }
                    });
                } catch (Throwable t) {
                    logger.warn(
                            "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                            AbstractChannel.this, t);
                    closeForcibly();
                    closeFuture.setClosed();
                    safeSetFailure(promise, t);
                }
            }
        }	

```

首先通过`isRegistered()` 判断该 Channel 是否已经注册到 EventLoop 中；  

通过 `eventLoop.inEventLoop()` 来判断当前线程是否为该 EventLoop 自身发起的，如果是，则调用 `register0()` 直接注册;

如果不是，说明该 EventLoop 中的线程此时没有执行权，则需要新建一个线程，单独封装一个 Task，而该 Task 的主要任务则是执行`register0()`。 

```java
		private void register0(ChannelPromise promise) {
            try {
                // 确保 Channel 处于 open
                if (!promise.setUncancellable() || !ensureOpen(promise)) {
                    return;
                }
                boolean firstRegistration = neverRegistered;

                // 真正的注册动作
                doRegister();

                neverRegistered = false;
                registered = true;        

                pipeline.invokeHandlerAddedIfNeeded();    
                safeSetSuccess(promise);        //设置注册结果为成功

                pipeline.fireChannelRegistered();

                if (isActive()) { 
                    //如果是首次注册,发起 pipeline 的 fireChannelActive
                    if (firstRegistration) {
                        pipeline.fireChannelActive();
                    } else if (config().isAutoRead()) {
                        beginRead();
                    }
                }
            } catch (Throwable t) {
                closeForcibly();
                closeFuture.setClosed();
                safeSetFailure(promise, t);
            }
        }

```

如果 Channel 处于 open 状态，则调用 `doRegister()` 方法完成注册，然后将注册结果设置为成功。最后判断如果是首次注册且处于激活状态，则发起 pipeline 的 `fireChannelActive()`。

```java
	protected void doRegister() throws Exception {
        boolean selected = false;
        for (;;) {
            try {
                // 注册到NIOEventLoop的Selector上
                selectionKey = javaChannel().register(eventLoop().selector, 0, this);
                return;
            } catch (CancelledKeyException e) {
                if (!selected) {
                    eventLoop().selectNow();
                    selected = true;
                } else {
                    throw e;
                }
            }
        }
    }

```

因为当前没有将一个ServerSocket绑定到一个address

```java
	if (isActive()) { 
		//如果是首次注册,发起 pipeline 的 fireChannelActive
		if (firstRegistration) {
			pipeline.fireChannelActive();
		} else if (config().isAutoRead()) {
			beginRead();
		}
	}

```

```java
	public boolean isActive() {
        return this.javaChannel().socket().isBound();
    }

```

```java
	protected void doBeginRead() throws Exception {
        SelectionKey selectionKey = this.selectionKey;
        if (selectionKey.isValid()) {
            this.readPending = true;
            int interestOps = selectionKey.interestOps();
            if ((interestOps & this.readInterestOp) == 0) {
                selectionKey.interestOps(interestOps | this.readInterestOp);
            }

        }
    }

```

这里将selectionKey的监听操作设置为之前构造NioServerSocketChannel设置的SelectionKey.OP_ACCEPT

##### he doBind0()

追溯doBind0()的实现，我们可以发现会调用初始化时NioMessageUnsafe的bind方法

```java
@Override
public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
    // ...
    boolean wasActive = isActive();
    // ...
    doBind(localAddress);

    if (!wasActive && isActive()) {
        invokeLater(new Runnable() {
            @Override
            public void run() {
                pipeline.fireChannelActive();
            }
        });
    }
    safeSetSuccess(promise);
}

```

doBind(localAddress) 调用JDK的代码，实现了端口绑定

```
protected void doBind(SocketAddress localAddress) throws Exception {
    if (PlatformDependent.javaVersion() >= 7) {
        //noinspection Since15
        javaChannel().bind(localAddress, config.getBacklog());
    } else {
        javaChannel().socket().bind(localAddress, config.getBacklog());
    }
}

```

绑定后isActice()返回true，fireChannelActive() 被调用。

## 4 内存管理

### 4.1 ByteBuffer、ByteBuf

为了减少频繁I/O操作，引进了Buffer的概念，把突发的大数量较小规模的 I/O 整理成平稳的小数量较大规模的 I/O 。Java NIO封装了ByteBuffer组件。ByteBuffer具有4个重要的属性：mark、position、limit、capacity ，以及两个重要的方法clear()、flip()

1. position:读写指针，代表当前读或写操作的位置，这个值总是小于等于limit的。
2. mark：在使用ByteBuffer的过程中，如果想要记住当前的position，则会将当前的position值给mark，需要恢复的时候，再将mark的值给position。
3. capacity：代表这块内存区域的大小。
4. limit：初始的Buffer中，limit和capacity的值是相等的，通常在clear操作和flip操作的时候会对这个值进行操作，在clear操作的时候会将这个值和capacity的值设置为相等，当flip的时候会将当前的position的值给limit，我们可以总结在写的时候，limit的值代表最大的可写位置，在读的时候，limit的值代表最大的可读位置。

![](/png/netty/ByteBuffer.png)

在写操作之前调用clear()

```java
public final Buffer clear() { 
	position = 0; //设置当前下标为0
	limit = capacity; //设置写越界位置与和Buffer容量相同
	mark = -1; //取消标记
	return this; 
} 

```

在读操作之前调用flip()

```java
public final Buffer flip() { 
	limit = position; 
	position = 0; 
	mark = -1; 
	return this; 
 }  

```

ByteBuffer具有以下缺陷

- ByteBuffer长度固定，一旦分配完成，它的容量不能动态扩展和收缩，当需要编码的POJO对象大于ByteBuffer的容量时，会发生索引越界异常；
- ByteBuffer只有一个标识位控的指针position，读写的时候需要手工调用 flip() 和 clear() 等；
- ByteBuffer的API功能有限，一些高级和实用的特性它不支持，需要使用者自己编程实现。

Netty为了解决ByteBuffer的缺陷，重写了一个新的数据接口ByteBuf。 与ByteBuffer相比，ByteBuf提供了两个指针 readerIndex 和 writeIndex 来分别指向读的位置和写的位置，不需要每次为读写做准备，直接设置读写指针进行读写操作即可。 

![](/png/netty/读写中间状态的Buffer.png)

这是中间状态的Buffer，可以通过调用discardReadBytes方法来回收已读区域

![](/png/netty/discardReadBytes.png)

通过clear方法清楚指针状态

![](/png/netty/clear后的Buffer.png)

对比ByteBuffer，使用ByteBuf读的时候仅仅依赖readerIndex指针，写的时候仅仅依赖writerIndex指针，不需每次读写之前调用对应的方法，而且没有必须一次读完的限制。 

### 4.2 内存管理

#### 4.2.1 零拷贝

当JVM堆内存上的数据需要和IO设备进行I/O操作时，会将JVM堆上所维护的byte[]拷贝至堆外内存（一般是通过C/C++分配的内存），然后堆外内存直接和IO设备交互。这是因为**JVM需要进行GC，如果IO设备直接和JVM堆上的数据进行交互，这个时候JVM进行了GC，那么有可能会导致没有被回收的数据进行了压缩，位置被移动到了连续的存储区域，这样会导致正在进行的IO操作相关的数据全部乱套**。

NIO可以使用native 函数库直接分配堆外内存，然后通过一个存储在堆上的DirectByteBuffer 对象作为这块内存的引用进行操作，避免了在Java堆和Native堆中来回复制数据。 

#### 4.2.2 内存泄漏

从堆中分配的缓冲区HeapByteBuffer为普通的Java对象，生命周期与普通的Java对象一样，当不再被引用时，Buffer对象会被回收。而直接缓冲区（DirectByteBuffer）为堆外内存，并不在Java堆中，也不能被JVM垃圾回收。由于直接缓冲区在JVM里被包装进Java对象DirectByteBuffer中，当它的包装类被垃圾回收时，会调用相应的JNI方法释放堆外内存，所以堆外内存的释放也依赖于JVM中DirectByteBuffer对象的回收。

由于垃圾回收本身成本较高，一般JVM在堆内存未耗尽时，不会进行垃圾回收操作。如果不断分配本地内存，堆内存很少使用，那么JVM就不需要执行GC，DirectByteBuffer对象们就不会被回收，这时候堆内存充足，但本地内存可能已经使用光了，再次尝试分配本地内存就会出现OutOfMemoryError，那程序就直接崩溃了。

#### 4.2.3 ByteBuf引用计数

```java
public interface ReferenceCounted {
    int refCnt();

    ReferenceCounted retain();

    ReferenceCounted retain(int var1);

    ReferenceCounted touch();

    ReferenceCounted touch(Object var1);

    boolean release();

    boolean release(int var1);
}

```

ByteBuf扩展了ReferenceCountered接口 ，这个接口定义的功能主要是引用计数。

当 ByteBuf 引用+1的时候，需要调用 retain() 来让refCnt + 1，当Buffer引用数-1的时候需要调用 release() 来让 refCnt - 1。当refCnt变为0的时候Netty为pooled和unpooled的不同buffer提供了不同的实现，通常对于非内存池的用法，Netty把Buffer的内存回收交给了垃圾回收器，对于内存池的用法，Netty对内存的回收实际上是回收到内存池内，以提供下一次的申请所使用。

#### 4.2.4 池化

如果对于Buffer的使用都基于直接内存实现的话，将会大大提高I/O效率，然而直接内存空间的申请比堆内存要消耗更高的性能。

因此Netty结合引用计数实现了PolledBuffer，即池化的用法，当引用计数等于0的时候，Netty将Buffer回收至池中，在下一次申请Buffer的时刻会被复用。 

堆内存和直接内存的池化实现分别是PooledHeapByteBuf和PooledDirectByteBuf，在各自的实现中都维护着一个Recycler 。Recycler是一个抽象类，向外部提供了两个公共方法get和recycle分别用于从对象池中获取对象和回收对象。

以PooledHeapByteBuf为例，新建PooledHeapByteBuf对象时

```java
	static PooledHeapByteBuf newInstance(int maxCapacity) {
        PooledHeapByteBuf buf = (PooledHeapByteBuf)RECYCLER.get();
        buf.reuse(maxCapacity);
        return buf;
    }

```

当Buffer引用数 -1时

```java
	public boolean release(int decrement) {
        return this.release0(ObjectUtil.checkPositive(decrement, "decrement"));
    }

    private boolean release0(int decrement) {
        int oldRef = refCntUpdater.getAndAdd(this, -decrement);
        if (oldRef == decrement) {
            this.deallocate();
            return true;
        } else if (oldRef >= decrement && oldRef - decrement <= oldRef) {
            return false;
        } else {
            refCntUpdater.getAndAdd(this, decrement);
            throw new IllegalReferenceCountException(oldRef, -decrement);
        }
    }

```

PooledByteBuf.class

```java
protected final void deallocate() {
    if (this.handle >= 0L) {
        long handle = this.handle;
        this.handle = -1L;
        this.memory = null;
        this.tmpNioBuf = null;
        this.chunk.arena.free(this.chunk, handle, this.maxLength, this.cache);
        this.chunk = null;
        this.recycle();
    }

}

private void recycle() {
    this.recyclerHandle.recycle(this);
}

```

### 4.3 半包读写

TCP是个"流"协议，所谓流，就是没有界限没有分割的一串数据。TCP会根据缓冲区的实际情况进行包划分，一个完整的包可能会拆分成多个包进行发送，也用可能把多个小包封装成一个大的数据包发送。这就是TCP粘包/拆包。

举个例子：假设操作系统已经接收到了三个包，如下：

![](/png/netty/流-拆包.png)

由于流传输的这个普通属性，在读取他们的时候将会存在很大的几率，这些数据会被分段成下面的几部分：

![](/png/netty/流-粘包.png)

也就是读取的数据有可能超过一个完整的数据包或者过多或者过少的半包。

因此，作为一个接收方，不管它是服务端还是客户端，都需要把接收到的数据整理成一个或多个有意义的并且能够被应用程序容易理解的数据。

**拆包方案：**

- **消息定长**，固定报文长度，不够空格补全，发送和接收方遵循相同的约定，这样即使粘包了通过接收方编程实现获取定长报文也能区分。
- **包尾添加特殊分隔符**，例如每条报文结束都添加回车换行符（例如FTP协议）或者指定特殊字符作为报文分隔符，接收方通过特殊分隔符切分报文区分。
- **将消息分为消息头和消息体**，消息头中包含表示信息的总长度（或者消息体长度）的字段

**Netty提供了几种解码器：**

- 定长解码器：FixedLengthFrameDecoder

```
ch.pipeline().addLast(new FixedLengthFrameDecoder(30));//设置定长解码器

```

- 特殊分隔符解码器：DelimiterBasedFrameDecoder

```
ByteBuf delimiter = Unpooled.copiedBuffer("&".getBytes());
//1024表示单条消息的最大长度，解码器在查找分隔符的时候，达到该长度还没找到的话会抛异常
ch.pipeline().addLast(new DelimiterBasedFrameDecoder(1024,delimiter));

```

- 基于包头不固定长度的解码器：LengthFieldBasedFrameDecoder

```java
/**
 * maxFrameLength：解码的帧的最大长度
 * lengthFieldOffset：长度属性的起始位（偏移位），包中存放有整个大数据包长度的字节，这段字节的其实位置
 * lengthFieldLength：长度属性的长度，即存放整个大数据包长度的字节所占的长度
 * lengthAdjustmen：长度调节值，在总长被定义为包含包头长度时，修正信息长度。
 * initialBytesToStrip：跳过的字节数，根据需要我们跳过lengthFieldLength个字节，以便接收端直接接受到不
                        含“长度属性”的内容
 */
ch.pipeline().addLast("decoder", new LengthFieldBasedFrameDecoder(MAX_FRAME_LENGTH, LENGTH_OFFSET, 
                                    LENGTH_LEN, LENGTH_ADJUGEMENT, INIT_BYTE_TO_STRIP));

```

**参考资料**

| 资料名称           | 来源                                        |
| ------------------ | ------------------------------------------- |
| 《Netty实战》      | 图书                                        |
| 《Netty权威指南》  | 图书                                        |
| Netty官网wiki      | https://netty.io/wiki/related-articles.html |
| 其他互联网资料链接 | 见最后                                      |


**参考链接：**

https://netty.io/index.html
https://blog.csdn.net/syc001/article/details/72841945
http://ifeve.com/%E8%B0%88%E8%B0%88netty%E7%9A%84%E7%BA%BF%E7%A8%8B%E6%A8%A1%E5%9E%8B/
https://www.cnblogs.com/wxd0108/p/6681627.html
https://my.oschina.net/plucury/blog/192577
https://blog.csdn.net/chenssy/article/details/78714003