# spark-network server端的消息发送和接收处理框架

## server端接收
在Spark的network 框架中，消息的服务方是类 `TransportServer`，该类的对象由TransportContext 所构造，

该类在启动的时候启动 Netty的 `ServerBootstrap`,用于接收网络请求， 并由TransportServer构造时的
RpcHandler appRpcHandler对象， 对channel 中的消息进行处理， 这里的RpcHandler时构造TransportServer的TransportContext
的rpcHandler，目前在代码中看到的主要有一下两种：

 1. NettyRpcEnv中transportContext 处理RPC请求
 2. NettyRpcEnv中的downloadContext 处理下载请求
 3. NettyBlockTransferService中的transportContext
 3. ExternalBlockStoreClient中构造clientFactory 用于获取rdd block
 3. ExternalShuffleService中的 ExternalBlockHandler： 服务器的RPC处理程序，它既可以为RDD块服务，也可以从执行程序进程外部洗牌块。
 处理注册executor和打开随机播放或磁盘持久化的RDD块。块使用“一对一”策略注册，这意味着每个transport-layer chunk相当于一个块。
 
 
### ServerBootStrap

```java
  private void init(String hostToBind, int portToBind) {

    IOMode ioMode = IOMode.valueOf(conf.ioMode());
    EventLoopGroup bossGroup =
      NettyUtils.createEventLoop(ioMode, conf.serverThreads(), conf.getModuleName() + "-server");
    EventLoopGroup workerGroup = bossGroup; // (1)

    bootstrap = new ServerBootstrap() // (2)
      .group(bossGroup, workerGroup) // (3)
      .channel(NettyUtils.getServerChannelClass(ioMode)) // (4)
      .option(ChannelOption.ALLOCATOR, pooledAllocator)  // (5)
      .option(ChannelOption.SO_REUSEADDR, !SystemUtils.IS_OS_WINDOWS)  // (5) 
      .childOption(ChannelOption.ALLOCATOR, pooledAllocator); // (6)

    this.metrics = new NettyMemoryMetrics(
      pooledAllocator, conf.getModuleName() + "-server", conf);

    if (conf.backLog() > 0) {
      bootstrap.option(ChannelOption.SO_BACKLOG, conf.backLog());
    }

    if (conf.receiveBuf() > 0) {
      bootstrap.childOption(ChannelOption.SO_RCVBUF, conf.receiveBuf());
    }

    if (conf.sendBuf() > 0) {
      bootstrap.childOption(ChannelOption.SO_SNDBUF, conf.sendBuf());
    }

    if (conf.enableTcpKeepAlive()) {
      bootstrap.childOption(ChannelOption.SO_KEEPALIVE, true);
    }

    bootstrap.childHandler(new ChannelInitializer<SocketChannel>() {  // (7)
      @Override
      protected void initChannel(SocketChannel ch) {
        logger.debug("New connection accepted for remote address {}.", ch.remoteAddress());

        RpcHandler rpcHandler = appRpcHandler;
        for (TransportServerBootstrap bootstrap : bootstraps) {
          rpcHandler = bootstrap.doBootstrap(ch, rpcHandler);
        }
        context.initializePipeline(ch, rpcHandler);
      }
    });

    InetSocketAddress address = hostToBind == null ?  // (8)
        new InetSocketAddress(portToBind): new InetSocketAddress(hostToBind, portToBind);
    channelFuture = bootstrap.bind(address);
    channelFuture.syncUninterruptibly();

    port = ((InetSocketAddress) channelFuture.channel().localAddress()).getPort();
    logger.debug("Shuffle server started on port: {}", port);
  }
```

 1. 初始化用于Acceptor的主"线程池"以及用于I/O工作的从"线程池"；
 2. 初始化ServerBootstrap实例， 此实例是netty服务端应用开发的入口，也是本篇介绍的重点， 下面我们会深入分析；
 3. 通过ServerBootstrap的group方法，设置（1）中初始化的主从"线程池"；
 4. 指定通道channel的类型，由于是服务端，故而是NioServerSocketChannel；
 5. 配置ServerSocketChannel的选项
 6. 配置子通道也就是SocketChannel的选项
 7. 设置子通道也就是SocketChannel的处理器， 其内部是实际业务开发的"主战场"（此处不详述，后面的系列会进行深入分析）
 8. 绑定并侦听某个端口
 
 
 ### TransportRequestHandler
 前面在TransportContext中有讲到，在TransportChannelHandler中，对于Reqeust类的请求会被转发到TransportRequestHandler
 处理，在TransportRequestHandler 中主要有以下几个对象：
 
  1. Channel channel 当前Netty Channel关联的channel
  2. TransportClient reverseClient 同一个请求信道可以向requester通信的client对象
  3. RpcHandler rpcHandler 处理rpc请求的handler，为初始化TransportContext时的handler
  4. StreamManager streamManager 处理StreamRequest的StreamManager
  
在TransportRequestHandler中，没有处理Chunk请求的方法，ChunkRequest由ChunkFetchRequestHandler处理

#### 主要的method

```java
  private void processStreamRequest(final StreamRequest req) {
    if (logger.isTraceEnabled()) {
      logger.trace("Received req from {} to fetch stream {}", getRemoteAddress(channel),
        req.streamId);
    }

    long chunksBeingTransferred = streamManager.chunksBeingTransferred();
    if (chunksBeingTransferred >= maxChunksBeingTransferred) {
      logger.warn("The number of chunks being transferred {} is above {}, close the connection.",
        chunksBeingTransferred, maxChunksBeingTransferred);
      channel.close();
      return;
    }
    ManagedBuffer buf;
    try {
      buf = streamManager.openStream(req.streamId);
    } catch (Exception e) {
      logger.error(String.format(
        "Error opening stream %s for request from %s", req.streamId, getRemoteAddress(channel)), e);
      respond(new StreamFailure(req.streamId, Throwables.getStackTraceAsString(e)));
      return;
    }

    if (buf != null) {
      streamManager.streamBeingSent(req.streamId);
      respond(new StreamResponse(req.streamId, buf.size(), buf)).addListener(future -> {
        streamManager.streamSent(req.streamId);
      });
    } else {
      // org.apache.spark.repl.ExecutorClassLoader.STREAM_NOT_FOUND_REGEX should also be updated
      // when the following error message is changed.
      respond(new StreamFailure(req.streamId, String.format(
        "Stream '%s' was not found.", req.streamId)));
    }
  }

  private void processRpcRequest(final RpcRequest req) {
    try {
      rpcHandler.receive(reverseClient, req.body().nioByteBuffer(), new RpcResponseCallback() {
        @Override
        public void onSuccess(ByteBuffer response) {
          respond(new RpcResponse(req.requestId, new NioManagedBuffer(response)));
        }

        @Override
        public void onFailure(Throwable e) {
          respond(new RpcFailure(req.requestId, Throwables.getStackTraceAsString(e)));
        }
      });
    } catch (Exception e) {
      logger.error("Error while invoking RpcHandler#receive() on RPC id " + req.requestId, e);
      respond(new RpcFailure(req.requestId, Throwables.getStackTraceAsString(e)));
    } finally {
      req.body().release();
    }
  }

  /**
   * Upload 类请求较为特殊，客户端的RPC请求携带需要用stream接收的数据
   * 通过wrappedCallback包装，携带RPC 回调的方法响应client的upload请求
   * 参见 ${RpcHandler.receiveStream()}
   * 
   * Handle a request from the client to upload a stream of data.
   */
  private void processStreamUpload(final UploadStream req) {
    assert (req.body() == null);
    try {
      RpcResponseCallback callback = new RpcResponseCallback() {
        @Override
        public void onSuccess(ByteBuffer response) {
          respond(new RpcResponse(req.requestId, new NioManagedBuffer(response)));
        }

        @Override
        public void onFailure(Throwable e) {
          respond(new RpcFailure(req.requestId, Throwables.getStackTraceAsString(e)));
        }
      };
      TransportFrameDecoder frameDecoder = (TransportFrameDecoder)
          channel.pipeline().get(TransportFrameDecoder.HANDLER_NAME);
      ByteBuffer meta = req.meta.nioByteBuffer();
      StreamCallbackWithID streamHandler = rpcHandler.receiveStream(reverseClient, meta, callback);
      if (streamHandler == null) {
        throw new NullPointerException("rpcHandler returned a null streamHandler");
      }
      StreamCallbackWithID wrappedCallback = new StreamCallbackWithID() {
        @Override
        public void onData(String streamId, ByteBuffer buf) throws IOException {
          streamHandler.onData(streamId, buf);
        }

        @Override
        public void onComplete(String streamId) throws IOException {
           try {
             streamHandler.onComplete(streamId);
             callback.onSuccess(ByteBuffer.allocate(0));
           } catch (Exception ex) {
             IOException ioExc = new IOException("Failure post-processing complete stream;" +
               " failing this rpc and leaving channel active", ex);
             callback.onFailure(ioExc);
             streamHandler.onFailure(streamId, ioExc);
           }
        }

        @Override
        public void onFailure(String streamId, Throwable cause) throws IOException {
          callback.onFailure(new IOException("Destination failed while reading stream", cause));
          streamHandler.onFailure(streamId, cause);
        }

        @Override
        public String getID() {
          return streamHandler.getID();
        }
      };
      if (req.bodyByteCount > 0) {
        StreamInterceptor<RequestMessage> interceptor = new StreamInterceptor<>(
          this, wrappedCallback.getID(), req.bodyByteCount, wrappedCallback);
        frameDecoder.setInterceptor(interceptor);
      } else {
        wrappedCallback.onComplete(wrappedCallback.getID());
      }
    } catch (Exception e) {
      logger.error("Error while invoking RpcHandler#receive() on RPC id " + req.requestId, e);
      respond(new RpcFailure(req.requestId, Throwables.getStackTraceAsString(e)));
      // We choose to totally fail the channel, rather than trying to recover as we do in other
      // cases.  We don't know how many bytes of the stream the client has already sent for the
      // stream, it's not worth trying to recover.
      channel.pipeline().fireExceptionCaught(e);
    } finally {
      req.meta.release();
    }
  }

  // OneWayMessage 失败时，不需要额外处理，直接释放message data
  private void processOneWayMessage(OneWayMessage req) {
    try {
      rpcHandler.receive(reverseClient, req.body().nioByteBuffer());
    } catch (Exception e) {
      logger.error("Error while invoking RpcHandler#receive() for one-way message.", e);
    } finally {
      req.body().release();
    }
  }

  /**
   * Responds to a single message with some Encodable object. If a failure occurs while sending,
   * it will be logged and the channel closed.
   */
  private ChannelFuture respond(Encodable result) {
    SocketAddress remoteAddress = channel.remoteAddress();
    return channel.writeAndFlush(result).addListener(future -> {
      if (future.isSuccess()) {
        logger.trace("Sent result {} to client {}", result, remoteAddress);
      } else {
        logger.error(String.format("Error sending result %s to %s; closing connection",
          result, remoteAddress), future.cause());
        channel.close();
      }
    });
  }
```