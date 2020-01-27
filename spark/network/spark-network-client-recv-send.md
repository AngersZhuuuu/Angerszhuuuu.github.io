# spark-network client端的消息发送和接收处理框架

## 客户端发送
在Spark的network 框架中，消息的发送由 `TransportClient` 类完成，客户端的TransportClient是由
`TransportClientFactory` 根据接收的host 和 port 生成对应的Socket链接，返回TransportClient

### TransportClient
TransportClient 是一个获取预先协商的数据流中连续块的客户端，这个客户端的api旨在块数的读取大量的数据，
这些数据预先被切分成几kb或者几mb的连续的有顺序的数据块。 
此客户端处理从流中获取块的过程中，流的实际相关设置是在传输层的范围之外完成的。
提供的方便的方法“sendRPC”，使客户端和服务器之间能够进行同一层面的通信。

一个客户端可以被多个stream使用，但是一个stream只能被限制给一个客户端， 避免出现respond混乱无序的情况。
在这个类中主要有一下几个对象

 * private final Channel channel;  进行网络IO通信，读写链接的套件
 * private final TransportResponseHandler handler;  负责响应处理服务端呢传来的responds消息
 * @Nullable private String clientId;  不为空的唯一的客户端id
 * private volatile boolean timedOut;  客户端请求超时时间
 
 
客户端的请求对应[network 消息类型](spark-network-message.html) 中所讲的集中类型的消息，在客户端中
分别由以下类型的请求：

#### fetchChunk()
```
 /**
   * 格局提前沟通的streamId向远端获取一个块的数据
   * Requests a single chunk from the remote side, from the pre-negotiated streamId.
   *
   * 块的索引从0线上增加，可以多次获取同一个块，但是有的远端的数据流可能不支持。
   * Chunk indices go from 0 onwards. It is valid to request the same chunk multiple times, though
   * some streams may not support this.
   *
   * 假设一个远端的数据流stream只被一个TransportClient请求获取数据的时候，
   * 申请块的请求可能是同时都没有完成，返回的数据块的顺序是保证按照请求的顺序。
   * 这也对应了前面说的，一个stream 严格对应一个客户端。
   * Multiple fetchChunk requests may be outstanding simultaneously, and the chunks are guaranteed
   * to be returned in the same order that they were requested, assuming only a single
   * TransportClient is used to fetch the chunks.
   *
   * @param streamId 指向远端的StreamManager的标志符，这是在处理数据块钱服务端和客户端协商好的标志符.
   * @param chunkIndex 从0开始的数据块的索引
   * @param callback 处理fetch块失败or成功的监听回调函数.
   */
  public void fetchChunk(
      long streamId,
      int chunkIndex,
      ChunkReceivedCallback callback) {
    if (logger.isDebugEnabled()) {
      logger.debug("Sending fetch chunk request {} to {}", chunkIndex, getRemoteAddress(channel));
    }

    StreamChunkId streamChunkId = new StreamChunkId(streamId, chunkIndex);
    StdChannelListener listener = new StdChannelListener(streamChunkId) {
      @Override
      void handleFailure(String errorMsg, Throwable cause) {
        handler.removeFetchRequest(streamChunkId);
        callback.onFailure(chunkIndex, new IOException(errorMsg, cause));
      }
    };
    // 将请求添加几responds handle，针对respond进行处理
    handler.addFetchRequest(streamChunkId, callback);

    // 写入链接套接字，远端的server 端对请求进行处理，返回将会被responds handler处理
    channel.writeAndFlush(new ChunkFetchRequest(streamChunkId)).addListener(listener);
  }
```

### stream()

```
  /**
   * 向远端请求数据流
   * Request to stream the data with the given stream ID from the remote end.
   *
   * @param streamId The stream to fetch.
   * @param callback Object to call with the stream data.
   */
  public void stream(String streamId, StreamCallback callback) {
    StdChannelListener listener = new StdChannelListener(streamId) {
      @Override
      void handleFailure(String errorMsg, Throwable cause) throws Exception {
        callback.onFailure(streamId, new IOException(errorMsg, cause));
      }
    };
    if (logger.isDebugEnabled()) {
      logger.debug("Sending stream request for {} to {}", streamId, getRemoteAddress(channel));
    }

    // Need to synchronize here so that the callback is added to the queue and the RPC is
    // written to the socket atomically, so that callbacks are called in the right order
    // when responses arrive.
    synchronized (this) {
      handler.addStreamCallback(streamId, callback);
      channel.writeAndFlush(new StreamRequest(streamId)).addListener(listener);
    }
  }
```

### sendRpc()
```
 /**
   * 向远端的RpcHandler 发送一个不透明的rpc消息，回调方法处理成功or失败。
   * Sends an opaque message to the RpcHandler on the server-side. The callback will be invoked
   * with the server's response or upon any failure.
   *
   * @param message 传递的消息，NIO的ByteBuffer.
   * @param callback 处理RPC回应的回调函数
   * @return The RPC's id.
   */
  public long sendRpc(ByteBuffer message, RpcResponseCallback callback) {
    if (logger.isTraceEnabled()) {
      logger.trace("Sending RPC to {}", getRemoteAddress(channel));
    }

    long requestId = requestId();
    // 添加回调方法到responds handler中去
    handler.addRpcRequest(requestId, callback);

    RpcChannelListener listener = new RpcChannelListener(requestId, callback);
    channel.writeAndFlush(new RpcRequest(requestId, new NioManagedBuffer(message)))
      .addListener(listener);

    return requestId;
  }
```


### uploadStream()
```
/**
   * 给远端的数据流传输数据，和前面的 stream()不同的是这个是传输数据，不是去从远端接收数据
   * Send data to the remote end as a stream.  This differs from stream() in that this is a request
   * to *send* data to the remote end, not to receive it from the remote.
   *
   * @param meta meta data associated with the stream, which will be read completely on the
   *             receiving end before the stream itself.
   * @param data this will be streamed to the remote end to allow for transferring large amounts
   *             of data without reading into memory.
   * @param callback handles the reply -- onSuccess will only be called when both message and data
   *                 are received successfully.
   */
  public long uploadStream(
      ManagedBuffer meta,
      ManagedBuffer data,
      RpcResponseCallback callback) {
    if (logger.isTraceEnabled()) {
      logger.trace("Sending RPC to {}", getRemoteAddress(channel));
    }

    long requestId = requestId();
    handler.addRpcRequest(requestId, callback);

    RpcChannelListener listener = new RpcChannelListener(requestId, callback);
    channel.writeAndFlush(new UploadStream(requestId, meta, data)).addListener(listener);

    return requestId;
  }
```
 
### sendRpcSync()

同步的方式发送RPC请求， 需要在一个指定的时间内获得返回结果。
```
/**
   * Synchronously sends an opaque message to the RpcHandler on the server-side, waiting for up to
   * a specified timeout for a response.
   */
  public ByteBuffer sendRpcSync(ByteBuffer message, long timeoutMs) {
    final SettableFuture<ByteBuffer> result = SettableFuture.create();

    sendRpc(message, new RpcResponseCallback() {
      @Override
      public void onSuccess(ByteBuffer response) {
        try {
          ByteBuffer copy = ByteBuffer.allocate(response.remaining());
          copy.put(response);
          // flip "copy" to make it readable
          copy.flip();
          result.set(copy);
        } catch (Throwable t) {
          logger.warn("Error in responding PRC callback", t);
          result.setException(t);
        }
      }

      @Override
      public void onFailure(Throwable e) {
        result.setException(e);
      }
    });

    try {
      return result.get(timeoutMs, TimeUnit.MILLISECONDS);
    } catch (ExecutionException e) {
      throw Throwables.propagate(e.getCause());
    } catch (Exception e) {
      throw Throwables.propagate(e);
    }
  }
``` 
 
### send()

```
  /**
   * 向远端的RpcHandler发送RPC请求，但是不需要返回，所以方法中不需要等待返回和添加监听器和回调函数
   * Sends an opaque message to the RpcHandler on the server-side. No reply is expected for the
   * message, and no delivery guarantees are made.
   *
   * @param message The message to send.
   */
  public void send(ByteBuffer message) {
    channel.writeAndFlush(new OneWayMessage(new NioManagedBuffer(message)));
  }

```


### TransportResponseHandler
TransportResponseHandler 是TransportClient 客户端处理响应消息的类， 其构造函数较为简单，需要的参数是对应链接的Channel的网络套接字。
其中有如下几个比较重要的对象：

 - outstandingFetches：ConcurrentHashMap of client `fetchChunk()` 时的StreamChunkId 和 对应的回调函数
 - outstandingRpcs：ConcurrentHashMap of client `sendRpc()` 时的 RpcId 和 对应的回调函数
 - streamCallbacks：Queue of client `stream()` 时的StreamId 和 对应的回调函数
 - timeOfLastRequestNs: 记录handle 上一次的请求时间

查看前面的TransportClient的API 可以发现，其在做异步请求的时候，会将对应的请求id 和 回调函数添加进 respond  handler的
存储对象中去，当处理（`TransportResponseHandler.handle()`）结束chunk rpc stream 三种请求方式分别对应下面6个api：

 - addFetchRequest()
 - removeFetchRequest()
 - addRpcRequest()
 - removeRpcRequest()
 - addStreamCallback()
 - deactivateStream()
 
 在`TransportResponseHandler`类中，处理各个异步请求的回调在其方法`handle()`中：
 
```
  @Override
  public void handle(ResponseMessage message) throws Exception {
  // 返回的是成功的message时，调用回调的onSuccess(), 失败时调用 onFailure()
    if (message instanceof ChunkFetchSuccess) {
      ChunkFetchSuccess resp = (ChunkFetchSuccess) message;
      ChunkReceivedCallback listener = outstandingFetches.get(resp.streamChunkId);
      if (listener == null) {
        // listener 为空， 没有回调， 直接将body（MessageBuffer）释放
        logger.warn("Ignoring response for block {} from {} since it is not outstanding",
          resp.streamChunkId, getRemoteAddress(channel));
        resp.body().release();
      } else {
        // 当有设置回调函数的时候，调用对应的回调函数，冰将MessageBuffer释放
        outstandingFetches.remove(resp.streamChunkId);
        listener.onSuccess(resp.streamChunkId.chunkIndex, resp.body());
        resp.body().release();
      }
    } else if (message instanceof ChunkFetchFailure) {
      ChunkFetchFailure resp = (ChunkFetchFailure) message;
      ChunkReceivedCallback listener = outstandingFetches.get(resp.streamChunkId);
      if (listener == null) {
        logger.warn("Ignoring response for block {} from {} ({}) since it is not outstanding",
          resp.streamChunkId, getRemoteAddress(channel), resp.errorString);
      } else {
        outstandingFetches.remove(resp.streamChunkId);
        listener.onFailure(resp.streamChunkId.chunkIndex, new ChunkFetchFailureException(
          "Failure while fetching " + resp.streamChunkId + ": " + resp.errorString));
      }
    } else if (message instanceof RpcResponse) {
    // RPC使用的时NIO格式信息数据
      RpcResponse resp = (RpcResponse) message;
      RpcResponseCallback listener = outstandingRpcs.get(resp.requestId);
      if (listener == null) {
        logger.warn("Ignoring response for RPC {} from {} ({} bytes) since it is not outstanding",
          resp.requestId, getRemoteAddress(channel), resp.body().size());
      } else {
        outstandingRpcs.remove(resp.requestId);
        // 因为rpc请求可能会继续作出别的请求，在这里 try finally包住，释放数据
        try {
          listener.onSuccess(resp.body().nioByteBuffer());
        } finally {
          resp.body().release();
        }
      }
    } else if (message instanceof RpcFailure) {
      RpcFailure resp = (RpcFailure) message;
      RpcResponseCallback listener = outstandingRpcs.get(resp.requestId);
      if (listener == null) {
        logger.warn("Ignoring response for RPC {} from {} ({}) since it is not outstanding",
          resp.requestId, getRemoteAddress(channel), resp.errorString);
      } else {
        outstandingRpcs.remove(resp.requestId);
        listener.onFailure(new RuntimeException(resp.errorString));
      }
    } else if (message instanceof StreamResponse) {
    // stream 的方法略为不同，其需要使用TransportFrameDecoder去处理
      StreamResponse resp = (StreamResponse) message;
      Pair<String, StreamCallback> entry = streamCallbacks.poll();
      if (entry != null) {
        StreamCallback callback = entry.getValue();
        if (resp.byteCount > 0) {
          StreamInterceptor<ResponseMessage> interceptor = new StreamInterceptor<>(
            this, resp.streamId, resp.byteCount, callback);
          try {
            TransportFrameDecoder frameDecoder = (TransportFrameDecoder)
              channel.pipeline().get(TransportFrameDecoder.HANDLER_NAME);
            frameDecoder.setInterceptor(interceptor);
            streamActive = true;
          } catch (Exception e) {
            logger.error("Error installing stream handler.", e);
            deactivateStream();
          }
        } else {
          try {
            callback.onComplete(resp.streamId);
          } catch (Exception e) {
            logger.warn("Error in stream handler onComplete().", e);
          }
        }
      } else {
        logger.error("Could not find callback for StreamResponse.");
      }
    } else if (message instanceof StreamFailure) {
      StreamFailure resp = (StreamFailure) message;
      Pair<String, StreamCallback> entry = streamCallbacks.poll();
      if (entry != null) {
        StreamCallback callback = entry.getValue();
        try {
          callback.onFailure(resp.streamId, new RuntimeException(resp.error));
        } catch (IOException ioe) {
          logger.warn("Error in stream failure handler.", ioe);
        }
      } else {
        logger.warn("Stream failure with unknown callback: {}", resp.error);
      }
    } else {
      throw new IllegalStateException("Unknown response type: " + message.type());
    }
  }

```


其中需要注意的点是

 - RpcResponseCallback 如果你需要在回调函数外去使用对应的message中的数据，
 因为在调用完回调函数，对应的数据被release， 需要copy一份数据
 - ChunkResponseCallback 如果想在回调函数外使用数据，需要call MnanagerBuffer的 retain方法
 
 上述的差异是因为，两个回调类的回调方法的参数不一样导致的 rpc 回调使用的参数是ByteBuffer，是直接的数据，chunk 使用的是ManagerBuffer
