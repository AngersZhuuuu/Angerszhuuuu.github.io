
# Spark RPC Frame 实现的方式
在前的部分中，讲到Spark的network的底层基于Netty的信息传输框架，我们知道在Spark中，Executor， Driver 是通过
各种个样的RPC message来保持通信的，接下来阐述怎噩梦实现RPC的小时传递和处理：

## 消息的包装发送
Spark 发送RPC调用`RpcEndpointRef.ask()/send()`等方法，在其继承类NettyRpcEndpointRef中我们可以看到
```java
  override def askAbortable[T: ClassTag](
      message: Any, timeout: RpcTimeout): AbortableRpcFuture[T] = {
    nettyEnv.askAbortable(new RequestMessage(nettyEnv.address, this, message), timeout)
  }

  override def ask[T: ClassTag](message: Any, timeout: RpcTimeout): Future[T] = {
    askAbortable(message, timeout).toFuture
  }

  override def send(message: Any): Unit = {
    require(message != null, "Message is null")
    nettyEnv.send(new RequestMessage(nettyEnv.address, this, message))
  }
```
在NettyRpcEndpointRef中将RPC消息包装成RequestMessage，在`NettyRpcEnv.askAbortable()`
方法中包装成`RpcOutboxMessage`,然后调用`NettyRpcenv.postToOutbox()` 方法。
在`postToOutbox()`方法中调用`RpcOutboxMessage.sendWith()`方法，使用对应的
TransportClient将消息发送出去， 在前面介绍TransportClient中我们讲到了RPC message发送的几个方法， `sendRpc()`,`sendRpcSync()`等，可以看到传递的参数时
ByteBuffer message，传递的参数是数据类型ByteBuffer
```java
  public long sendRpc(ByteBuffer message, RpcResponseCallback callback) {
    if (logger.isTraceEnabled()) {
      logger.trace("Sending RPC to {}", getRemoteAddress(channel));
    }

    long requestId = requestId();
    handler.addRpcRequest(requestId, callback);

    RpcChannelListener listener = new RpcChannelListener(requestId, callback);
    channel.writeAndFlush(new RpcRequest(requestId, new NioManagedBuffer(message)))
      .addListener(listener);

    return requestId;
  }
```
可以看到在发送的时候使用NioManagedBuffer 将详细实体包装，这些信息会经过MessageEncoder加密，写到Channel中，
在接收段Channel接收到数据时被MessageDecoder解密成对应的消息类型，RpcRequest被TransportChannelHandler处理，
在`TransportChannelHandler.processRpcMessage()`中，`req.body().nioByteBuffer()`,获取到包装成RpcRequest前
的数据.
```java
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
```
上述的`rpcHandler`在RPC的代码框架中指的是`NettyRpcHandler`，在`NettyRpcHandler`的`receive()`类的方法中，会将信息进行装箱，转换成
InboxMessage类型，然后投递给Dispatcher：

```java
override def receive(
      client: TransportClient,
      message: ByteBuffer,
      callback: RpcResponseCallback): Unit = {
    val messageToDispatch = internalReceive(client, message)
    dispatcher.postRemoteMessage(messageToDispatch, callback)
  }
```
Dispacther中的MessageLoop对象会对不断投递进来的消息进行处理，MessageLoop中的`receiveLoopRunnable`对象启动，调用链
 ```java
 MessageLoop.receiveLoop() ->  Inbox.process()  -> RpcEndpoint.receiveAndReply()/receive()
```
在RpcEndpoint实体对象中，调用上述方法拿到的是最终的消息类容，然后各个终端针对不同的消息内容进行处理，例如：
```java
  override def receive: PartialFunction[Any, Unit] = {
    case RegisteredExecutor =>
      logInfo("Successfully registered with driver")
      try {
        executor = new Executor(executorId, hostname, env, userClassPath, isLocal = false)
        driver.get.send(LaunchedExecutor(executorId))
      } catch {
        case NonFatal(e) =>
          exitExecutor(1, "Unable to create executor due to " + e.getMessage, e)
      }

    case RegisterExecutorFailed(message) =>
      exitExecutor(1, "Slave registration failed: " + message)

    case LaunchTask(data) =>
      if (executor == null) {
        exitExecutor(1, "Received LaunchTask command but executor was null")
      } else {
        val taskDesc = TaskDescription.decode(data.value)
        logInfo("Got assigned task " + taskDesc.taskId)
        taskResources(taskDesc.taskId) = taskDesc.resources
        executor.launchTask(this, taskDesc)
      }

    case KillTask(taskId, _, interruptThread, reason) =>
      if (executor == null) {
        exitExecutor(1, "Received KillTask command but executor was null")
      } else {
        executor.killTask(taskId, interruptThread, reason)
      }

    case StopExecutor =>
      stopping.set(true)
      logInfo("Driver commanded a shutdown")
      // Cannot shutdown here because an ack may need to be sent back to the caller. So send
      // a message to self to actually do the shutdown.
      self.send(Shutdown)

    case Shutdown =>
      stopping.set(true)
      new Thread("CoarseGrainedExecutorBackend-stop-executor") {
        override def run(): Unit = {
          // executor.stop() will call `SparkEnv.stop()` which waits until RpcEnv stops totally.
          // However, if `executor.stop()` runs in some thread of RpcEnv, RpcEnv won't be able to
          // stop until `executor.stop()` returns, which becomes a dead-lock (See SPARK-14180).
          // Therefore, we put this line in a new thread.
          executor.stop()
        }
      }.start()

    case UpdateDelegationTokens(tokenBytes) =>
      logInfo(s"Received tokens of ${tokenBytes.length} bytes")
      SparkHadoopUtil.get.addDelegationTokens(tokenBytes, env.conf)
  }
```