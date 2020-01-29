# TransportContext

TransportContext 是Spark 网络通信模块重要的类，具体介绍其中的几个重要方法
其中包含构造`TransportServer`, `TransportClientFactory`的内容， 同时在分别构造服务端和客户端的时候，
设置Netty Channel的pipeline `org.apache.spark.network.server.TransportChannelHandler`

在Transport中TransportClient 提供两种通信协议`control-plane RPCs`和`data-plane "chunk fetching"`. 
处理RPCs不再TransportContext的控制范围内，而是使用客户端定制的Handler 比如 NettyEnvRpcHandler。
同时，他提供使用令0拷贝的stream数据层交互。
`TransportServer` 和 `TransportClientFactory` 都会为每一个通讯channel创建 `TransportChannelHandler`。
每一个`TransportChannelHandler`包含一个`TransportClient`,使服务端也可以向已存在的客户端发送消息。

![Client Server 消息传递](channel.png)


## 主要API


### createClientFactory()
初始化一个在返回客户端之前，使用给定TransportClientFactory的ClientFactory。
BootStraps将同步执行，并且必须成功运行才能创建客户端。

### createServer()

初始化TransportServer

### initializePipeline()

初始化一个client 或者 server 的Netty Channel Pipeline ， 这个pipeline可以 encodes/decodes 消息
并且又一个 `org.apache.spark.network.server.TransportChannelHandler`去处理请求或者响应
   
返回创建的TransportChannelHandler，其中包括一个可用于在此通道上通信的TransportClient。
TransportClient直接与ChannelHandler关联，以确保同一通道的所有用户获得相同的TransportClient对象。

```java
public TransportChannelHandler initializePipeline(
      SocketChannel channel,
      RpcHandler channelRpcHandler) {
    try {
      TransportChannelHandler channelHandler = createChannelHandler(channel, channelRpcHandler);
      ChunkFetchRequestHandler chunkFetchHandler =
        createChunkFetchHandler(channelHandler, channelRpcHandler);
      ChannelPipeline pipeline = channel.pipeline()
        // 发送消息的加密
        .addLast("encoder", ENCODER)
        .addLast(TransportFrameDecoder.HANDLER_NAME, NettyUtils.createFrameDecoder())
        // 接收消息解码
        .addLast("decoder", DECODER)
        .addLast("idleStateHandler",
          // 当一个channel 一段时间没有读写时间，触发超时事件
          new IdleStateHandler(0, 0, conf.connectionTimeoutMs() / 1000))
        // NOTE: Chunks are currently guaranteed to be returned in the order of request, but this
        // would require more logic to guarantee if this were not part of the same event loop.
        .addLast("handler", channelHandler);
      // chunk消息的特殊性，需要使用独立的EventLoopGroup去处理保证逻辑完整
      // 处理ChunkFetchRequest的独立线程池。这有助于限制在通过底层通道将
      // ChunkFetchRequest消息的响应写回客户机时阻塞的TransportServer工作线程
      // 的最大数量。
      // Use a separate EventLoopGroup to handle ChunkFetchRequest messages for shuffle rpcs.
      if (chunkFetchWorkers != null) {
        pipeline.addLast(chunkFetchWorkers, "chunkFetchHandler", chunkFetchHandler);
      }
      return channelHandler;
    } catch (RuntimeException e) {
      logger.error("Error while initializing Netty pipeline", e);
      throw e;
    }
  }

```
### createChannelHandler()

创建服务器端和客户端处理程序，用于处理请求消息和响应消息。虽然某些属性(如remoteAddress())可能还不可用，
但是预期通道已经成功创建。

### createChunkFetchHandler()
为ChunkFetchRequest消息创建专用的ChannelHandler。

## TransportChannelHandler

TransportChannelHandler 负责处理channel中接收到的消息， 主要有三个功能

 1. 将request和respond分发给RTransportRequestHandler或者TransportRespondHandler
 2. 重写acceptInboundMessage将ChunkFetchRequest抛给更上层的handler `ChunkFetchRequestHandler`

### 分发处理request和respond类型消息

```java
  @Override
  public void channelRead0(ChannelHandlerContext ctx, Message request) throws Exception {
    if (request instanceof RequestMessage) {
      requestHandler.handle((RequestMessage) request);
    } else if (request instanceof ResponseMessage) {
      responseHandler.handle((ResponseMessage) request);
    } else {
      ctx.fireChannelRead(request);
    }
  }
```

### acceptInboundMessage() 抛出 ChunkFetchRequest
```java
  /**
   * Overwrite acceptInboundMessage to properly delegate ChunkFetchRequest messages
   * to ChunkFetchRequestHandler.
   */
  @Override
  public boolean acceptInboundMessage(Object msg) throws Exception {
    if (msg instanceof ChunkFetchRequest) {
      return false;
    } else {
      return super.acceptInboundMessage(msg);
    }
  }
```

### userEventTriggered() 方法处理IdleStateHandler触发的超时事件
```java
/** Triggered based on events from an {@link io.netty.handler.timeout.IdleStateHandler}. */
  @Override
  public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
    if (evt instanceof IdleStateEvent) {
      IdleStateEvent e = (IdleStateEvent) evt;
      // See class comment for timeout semantics. In addition to ensuring we only timeout while
      // there are outstanding requests, we also do a secondary consistency check to ensure
      // there's no race between the idle timeout and incrementing the numOutstandingRequests
      // (see SPARK-7003).
      //
      // To avoid a race between TransportClientFactory.createClient() and this code which could
      // result in an inactive client being returned, this needs to run in a synchronized block.
      synchronized (this) {
        boolean hasInFlightRequests = responseHandler.numOutstandingRequests() > 0;
        boolean isActuallyOverdue =
          System.nanoTime() - responseHandler.getTimeOfLastRequestNs() > requestTimeoutNs;
        if (e.state() == IdleState.ALL_IDLE && isActuallyOverdue) {
          if (hasInFlightRequests) {
            String address = getRemoteAddress(ctx.channel());
            logger.error("Connection to {} has been quiet for {} ms while there are outstanding " +
              "requests. Assuming connection is dead; please adjust spark.network.timeout if " +
              "this is wrong.", address, requestTimeoutNs / 1000 / 1000);
            client.timeOut();
            ctx.close();
          } else if (closeIdleConnections) {
            // While CloseIdleConnections is enable, we also close idle connection
            client.timeOut();
            ctx.close();
          }
        }
      }
    }
    ctx.fireUserEventTriggered(evt);
  }

```