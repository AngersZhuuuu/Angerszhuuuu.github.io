

# RpcHandler

前面的内容可以看到，无论是client端还是服务端，实际处理消息的对象是RpcHandler的继承类，
不同的继承类有不同的功能类型，启动对应的TransportContext时的RpcHandler及为实际处理消息
的RpcHandler，主要有一下集中RpcHandler：

 1. NettyBlockRpcServer: 为请求open block提供服务，只需为每个请求块注册一个块。
    处理打开和上传任意块BlockManager块。打开的块以“一个对一个”的策略注册，这意味
    着每个传输层块相当于一个spark级别的shuffle块。
 2. NettyRpcHandler: Dispatcher将rpc请求分发给不同的endpoint，再一个端可能有多个rpc endpoint
 3. AuthRpcHandler: RPC处理程序，在委托给子RPC处理程序之前，使用Spark的auth协议执行身份验证。 auth相关不多说
 4. SaslRpcHandler: RPC处理程序，在委托给子RPC处理程序之前，sasl协议执行身份验证
 5. NoOpRpcHandler: 它只适用于不能接收rpc的客户端TransportContext
 6. ExternalBlockHandler: 服务器的RPC处理程序，它既可以为RDD块服务，也可以从Executor进程外部洗牌块
 
 
在这里我们重点学习的时NettyBlockRpcServer， NettyRpcHandler 和 ExternalBlockHandler，下面介绍RpcHandler的接口API

## API

接收一个单个的RPC请求，如果该方法有错误返回一个字符串格式的标砖RPC失败响应， `#receiveStream`和该方法
在同一个TransportClient上不可以并行
```java
  public abstract void receive(
      TransportClient client,
      ByteBuffer message,
      RpcResponseCallback callback);

```

接收单个的包含数据的RPC请求，并接收时当作字符串接收，该方法中发生错误时返回字符串格式的RPC失败消息。
StreamCallback#onData(String, ByteBuffer)中的失败导致整个channel失败，StreamCallback#onComplete(String)
会返回RpcFailure但是不会导致channel失败，channel还是active

```java
  public StreamCallbackWithID receiveStream(
      TransportClient client,
      ByteBuffer messageHeader,
      RpcResponseCallback callback) {
    throw new UnsupportedOperationException();
  }

```

返回包含所有客户端哪去stream数据的状态的StreamManager
```java
  public abstract StreamManager getStreamManager();
```

接收不需要reply的RPC消息
```java
  public void receive(TransportClient client, ByteBuffer message) {
    receive(client, message, ONE_WAY_CALLBACK);
  }
```

当client关联的channel活跃时触发调用
```java
  public void channelActive(TransportClient client) { }
```

当client关联的channel不活跃时触发调用，后续不会有请求从这个client发出
```java
  public void channelInactive(TransportClient client) { }
```

```java
  public void exceptionCaught(Throwable cause, TransportClient client) { }

```