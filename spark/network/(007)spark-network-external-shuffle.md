 
# External Shuffle Service
 
Spark系统在运行含shuffle过程的应用时，Executor进程除了运行task，还要负责写shuffle 数据，给其他Executor提供shuffle数据。
当Executor进程任务过重，导致GC而不能为其 他Executor提供shuffle数据时，会影响任务运行。
这里实际上是利用External Shuffle Service 来提升性能，External shuffle Service是长期存在于NodeManager进程中的一个辅助服务。
通过该服务 来抓取shuffle数据，减少了Executor的压力，在Executor GC的时候也不会影响其他 Executor的任务运行。

External Shuffle Service的类：

 * ExternalShuffleService  服务与Standalone 模式的Worker上
 * YarnShuffleService  服务于Yarn的NodeManager上


以Yarn模式为例，介绍External Shuffle Service怎么交换数据,在之前需要理清的点：

 1. 未开启External Shuffle Service，获取数据块的服务类为NettyBlockRpcServer
 2. 开启External Shuffle Service 获取数据的服务类为 ExternalShuffleService(Standalone mode,运行在每个worker上)/
 YarnShuffleService（yarn mode， 运行在每个NodeManager上）
 3. 未开启External Shuffle Service， Driver端数据的客户端为NettyBlockTransferService，Executor shuffle数据的客户端为NettyBlockTransferService
 4. 开启External Shuffle Service，Driver端数据的客户端为NettyBlockTransferService，Executor shuffle数据的客户端为ExternalBlockStoreClient
 
 

## ExternalBlockHandler

ExternalBlockHandler 类中构造ExternalBlockHandler对象，并以该RpcHandler对象构造TransportContext，并创建对应的
TransportServer作为服务端。下面我们重点介绍数据块请求消息的处理过程及 `ExternalBlockHandler`.
在介绍这些之前首先了解Block在各个节点上如何存储

### BlockManager 存储数据
BlockManager运行在每个节点上（包括Driver和Executor），提供对本地或远端节点上的内存、磁盘及堆外内存中Block的管理。
存储体系从狭义上来说指的就是BlockManager，从广义上来说，则包括整个Spark集群中的各个 BlockManager、BlockInfoManager、
DiskBlockManager、DiskStore、MemoryManager、MemoryStore、对集群中的所有BlockManager进行管理的BlockManagerMaster
及各个节点上对外提供Block上传与下载服务的BlockTransferService。

blockId 对应的文件关系, spark在本地存储block时，从设置的localDir列表中选择根据BlockId的hash值选取对应目录，然后通过hash值
选择子目录的id值，然后拼接blockId的值，成为该blockId的唯一FIleName，因为我们配置的localDirs通常是固定的，每个localDir下子
dir的数目也是固定的，所以每一个blockId都会对应一个唯一的File Path，这样保证无论是在一个程序服务内部还是使用External SHuffle 
Service， 只要知道BlockId，和对应的Executor的host，就可以准确的拿到对应的data。
```java
  public static File getFile(String[] localDirs, int subDirsPerLocalDir, String filename) {
    int hash = JavaUtils.nonNegativeHash(filename);
    String localDir = localDirs[hash % localDirs.length];
    int subDirId = (hash / localDirs.length) % subDirsPerLocalDir;
    return new File(createNormalizedInternedPathname(
        localDir, String.format("%02x", subDirId), filename));
  }
```

### BlockManager请求远程数据

#### Driver端获取远程数据
每个BlockManager存储的这些数据通过UploadBlock类型的RPC通知Driver端，这样Driver端能知道每个RDD对应的数据块的分布位置和BlockId
需要做数据shuffle的时候，driver端的task中携带要被获取的数据Block信息，然后根据Block的信息，调用BlockManager的相关方法：

 * getRemoteValue() 
 * getRemoteBytes() 被TaskResultGetter和TorrentBroadCast所调用，
 
 上述两个方法均调用 `getRemoteBlock()`,赘述 `getBlockBlock()`后面的调用栈，仅展示大概，直到和ExternalShuffleService有关：


```java
数据在本地
BlockManager.getRemoveBlcok() -> BlockManager.readDiskBlockFromSameHostExecutor() ->
 -> EncryptedManagedBuffer/FileSegmentManagedBuffer



数据在remote
BlockManager.getRemoveBlcok() -> BlockManager.fetchRemoteManagedBuffer() -> 
-> BlockTransferService.fetchBlockSync() -> BlockStoreClient.fetchBlocks()
```

在这个地方又一个区分点，在spark中由配置
`spark.shuffle.service.enabled`, 当其为true时，使用External shuffle， 当其为false时，使用直接链接executor的transfer service

NettyBlockTransferService 是继承ExternalStoreClient直接链接Executor获取数据的客户端，
ExternalShuffleService为继承ExternalStoreClient的External Shuffle Service的客户端，
上述两种BlockStoreClient获取数据的方法基本一致，下面以 NettyBlockTransferService 的实现方法为例，讲述获取数据的流程
```scala
   def fetchBlocks(
      host: String,
      port: Int,
      execId: String,
      blockIds: Array[String],
      listener: BlockFetchingListener,
      tempFileManager: DownloadFileManager): Unit = {
    logTrace(s"Fetch blocks from $host:$port (executor id $execId)")
    try {
      val blockFetchStarter = new RetryingBlockFetcher.BlockFetchStarter {
        override def createAndStart(blockIds: Array[String],
            listener: BlockFetchingListener): Unit = {
          try {
            val client = clientFactory.createClient(host, port)
            new OneForOneBlockFetcher(client, appId, execId, blockIds, listener,
              transportConf, tempFileManager).start()
          } catch {
            case e: IOException =>
              Try {
                driverEndPointRef.askSync[Boolean](IsExecutorAlive(execId))
              } match {
                case Success(v) if v == false =>
                  throw new ExecutorDeadException(s"The relative remote executor(Id: $execId)," +
                    " which maintains the block data to fetch is dead.")
                case _ => throw e
              }
          }
        }
      }

      val maxRetries = transportConf.maxIORetries()
      if (maxRetries > 0) {
        // Note this Fetcher will correctly handle maxRetries == 0; we avoid it just in case there's
        // a bug in this code. We should remove the if statement once we're sure of the stability.
        new RetryingBlockFetcher(transportConf, blockFetchStarter, blockIds, listener).start()
      } else {
        blockFetchStarter.createAndStart(blockIds, listener)
      }
    } catch {
      case e: Exception =>
        logError("Exception while beginning fetchBlocks", e)
        blockIds.foreach(listener.onBlockFetchFailure(_, e))
    }
  }
```

从上面可以看到数据获取的过程抛开Retry层面的代理和异常捕获， 使用的时OneForOneBlockFetcher的start()方法请求数据，，
在这里又一个参数比较重要，当我们使用高版本的spark 去低版本的External SHuffle Server去读取数据时，
一定要设置使用旧的协议.
```java
 /**
   * Whether to use the old protocol while doing the shuffle block fetching.
   * It is only enabled while we need the compatibility in the scenario of new spark version
   * job fetching blocks from old version external shuffle service.
   */
  public boolean useOldFetchProtocol() {
    return conf.getBoolean("spark.shuffle.useOldFetchProtocol", false);
  }
```

### ExternalBlockHandler 处理请求

旧的协议为`OpenBlocks`, 新的协议为`FetchShuffleBlocks`,这里不过多阐述其协议的区别，在ExternalBlockHandler中，针对这两种请求，
请求成功的状态下均会返回对应的StreamId， 并在StreamManager中注册对应的数据流；

#### ExternalShuffle Service 的 ExternalBlockHandler处理请求
```java
  protected void handleMessage(
      BlockTransferMessage msgObj,
      TransportClient client,
      RpcResponseCallback callback) {
    if (msgObj instanceof FetchShuffleBlocks || msgObj instanceof OpenBlocks) {
      final Timer.Context responseDelayContext = metrics.openBlockRequestLatencyMillis.time();
      try {
        int numBlockIds;
        long streamId;
        if (msgObj instanceof FetchShuffleBlocks) {
          FetchShuffleBlocks msg = (FetchShuffleBlocks) msgObj;
          checkAuth(client, msg.appId);
          numBlockIds = 0;
          if (msg.batchFetchEnabled) {
            numBlockIds = msg.mapIds.length;
          } else {
            for (int[] ids: msg.reduceIds) {
              numBlockIds += ids.length;
            }
          }
          streamId = streamManager.registerStream(client.getClientId(),
            new ShuffleManagedBufferIterator(msg), client.getChannel());
        } else {
          // For the compatibility with the old version, still keep the support for OpenBlocks.
          OpenBlocks msg = (OpenBlocks) msgObj;
          numBlockIds = msg.blockIds.length;
          checkAuth(client, msg.appId);
          streamId = streamManager.registerStream(client.getClientId(),
            new ManagedBufferIterator(msg), client.getChannel());
        }
       ··················
       ··················
    } else {
      throw new UnsupportedOperationException("Unexpected message: " + msgObj);
    }
  }
```

#### 直连Executor获取数据服务类NettyBlockRpcServer

```scala
  override def receive(
      client: TransportClient,
      rpcMessage: ByteBuffer,
      responseContext: RpcResponseCallback): Unit = {
    val message = BlockTransferMessage.Decoder.fromByteBuffer(rpcMessage)
    logTrace(s"Received request: $message")

    message match {
      case openBlocks: OpenBlocks =>
        val blocksNum = openBlocks.blockIds.length
        val blocks = (0 until blocksNum).map { i =>
          val blockId = BlockId.apply(openBlocks.blockIds(i))
          assert(!blockId.isInstanceOf[ShuffleBlockBatchId],
            "Continuous shuffle block fetching only works for new fetch protocol.")
          blockManager.getLocalBlockData(blockId)
        }
        val streamId = streamManager.registerStream(appId, blocks.iterator.asJava,
          client.getChannel)
        logTrace(s"Registered streamId $streamId with $blocksNum buffers")
        responseContext.onSuccess(new StreamHandle(streamId, blocksNum).toByteBuffer)

      case fetchShuffleBlocks: FetchShuffleBlocks =>
        val blocks = fetchShuffleBlocks.mapIds.zipWithIndex.flatMap { case (mapId, index) =>
          if (!fetchShuffleBlocks.batchFetchEnabled) {
            fetchShuffleBlocks.reduceIds(index).map { reduceId =>
              blockManager.getLocalBlockData(
                ShuffleBlockId(fetchShuffleBlocks.shuffleId, mapId, reduceId))
            }
          } else {
            val startAndEndId = fetchShuffleBlocks.reduceIds(index)
            if (startAndEndId.length != 2) {
              throw new IllegalStateException(s"Invalid shuffle fetch request when batch mode " +
                s"is enabled: $fetchShuffleBlocks")
            }
            Array(blockManager.getLocalBlockData(
              ShuffleBlockBatchId(
                fetchShuffleBlocks.shuffleId, mapId, startAndEndId(0), startAndEndId(1))))
          }
        }

        val numBlockIds = if (fetchShuffleBlocks.batchFetchEnabled) {
          fetchShuffleBlocks.mapIds.length
        } else {
          fetchShuffleBlocks.reduceIds.map(_.length).sum
        }

        val streamId = streamManager.registerStream(appId, blocks.iterator.asJava,
          client.getChannel)
        logTrace(s"Registered streamId $streamId with $numBlockIds buffers")
        responseContext.onSuccess(
          new StreamHandle(streamId, numBlockIds).toByteBuffer)

      case uploadBlock: UploadBlock =>
        // StorageLevel and ClassTag are serialized as bytes using our JavaSerializer.
        val (level, classTag) = deserializeMetadata(uploadBlock.metadata)
        val data = new NioManagedBuffer(ByteBuffer.wrap(uploadBlock.blockData))
        val blockId = BlockId(uploadBlock.blockId)
        logDebug(s"Receiving replicated block $blockId with level ${level} " +
          s"from ${client.getSocketAddress}")
        blockManager.putBlockData(blockId, data, level, classTag)
        responseContext.onSuccess(ByteBuffer.allocate(0))
    }
  }
```

### OneForOneBlockFetcher 处理stream 数据获取
前面讲到由`OneForOneBlockFetcher.start()`方法获取数据

```java
  /**
   * Begins the fetching process, calling the listener with every block fetched.
   * The given message will be serialized with the Java serializer, and the RPC must return a
   * {@link StreamHandle}. We will send all fetch requests immediately, without throttling.
   */
  public void start() {
    // 通过TransportClient发送RPC获取数据请求，由服务端确定返回StreamI， 这 和前面讲述的stream类
    // 数据的获取需要客户端和服务端协商好一致
    client.sendRpc(message.toByteBuffer(), new RpcResponseCallback() {
      @Override
      public void onSuccess(ByteBuffer response) {
        try {
          streamHandle = (StreamHandle) BlockTransferMessage.Decoder.fromByteBuffer(response);
          logger.trace("Successfully opened blocks {}, preparing to fetch chunks.", streamHandle);

          // Immediately request all chunks -- we expect that the total size of the request is
          // reasonable due to higher level chunking in [[ShuffleBlockFetcherIterator]].
          // 根据协商的StreamId等信息，one by one的获取连续的有序的Chunk数据
          // 当 downloadFileManager != null 的时候，使用stream的方式，否则使用fetchChunk的方式
          for (int i = 0; i < streamHandle.numChunks; i++) {
            if (downloadFileManager != null) {
              client.stream(OneForOneStreamManager.genStreamChunkId(streamHandle.streamId, i),
                new DownloadCallback(i));
            } else {
              client.fetchChunk(streamHandle.streamId, i, chunkCallback);
            }
          }
        } catch (Exception e) {
          logger.error("Failed while starting block fetches after success", e);
          failRemainingBlocks(blockIds, e);
        }
      }

      @Override
      public void onFailure(Throwable e) {
        logger.error("Failed while starting block fetches", e);
        failRemainingBlocks(blockIds, e);
      }
    });
  }
```

#### RDD 之间Shuffle交换数据

在Rdd由各种算子操作期间生成各种需要Shuffle交换数据的Shuffle RDD 类型， 例如：

 - ShuffleRDD
 - ShuffledRawRDD
 - CoGroupedRDD
 - SubtractedRDD
 
这里不细说这些不同类型的需要Shuffle 数据RDD的使用场景，RDD的方法`compute()`会计算得到结果RDD，
在上述这些RDD的`compute()`方法中，均调用`SparkEnv.get.shuffleManager.getReader()`方法，
已知在当前的Spark版本中，Shuffle Manager的实现类只有一种`SortShuffleManager`,查看其`getReader()`
方法, 这是在每个Executor的reduce task中被调用，生成一个reader去从前面的stage读取partition的数据
```scala
 /**
   * Get a reader for a range of reduce partitions (startPartition to endPartition-1, inclusive).
   * Called on executors by reduce tasks.
   */
  override def getReader[K, C](
      handle: ShuffleHandle,
      startPartition: Int,
      endPartition: Int,
      context: TaskContext,
      metrics: ShuffleReadMetricsReporter): ShuffleReader[K, C] = {
    val blocksByAddress = SparkEnv.get.mapOutputTracker.getMapSizesByExecutorId(
      handle.shuffleId, startPartition, endPartition)
    new BlockStoreShuffleReader(
      handle.asInstanceOf[BaseShuffleHandle[K, _, C]], blocksByAddress, context, metrics,
      shouldBatchFetch = canUseBatchFetch(startPartition, endPartition, context))
  }
```
`MapOutputTracker.getMapSizesByExecutorId()`方法会获得这一次的Shuffle需要的数据在map output阶段的对应信息
生成`BlockStoreShuffleReader`对象，在`BlockStoreShuffleReader`的`read()`方法中，生成`ShuffleBlockFetcherIterator`对象，
并将远程的数据Block转成Iterator对象，`CompletionIterator`，在map等相关集合操作中，有后台API自动调用`next()/hasNext()`等一些列继承的API
在`ShuffleeBlockFetcherIterator`中，使用的shuffleClient是 `BlockManager.shuffleClient`,该对象在使用ExternalShuffleService时，
是ExternalShuffleStoreClient，否则使用直接链接Executor获取数据的NettyBlockTransferService。两种client获取数据的方式基本一样，这里不作区分讨论
```java
  // Client to read other executors' blocks. This is either an external service, or just the
  // standard BlockTransferService to directly connect to other Executors.
  private[spark] val blockStoreClient = externalBlockStoreClient.getOrElse(blockTransferService)

```

在`ShuffleBlockFetcherIterator`中，获取数据的方式如下：

 1. 类初始化的时候调用initialize() 调用 fetchUpToMaxBytes(), 获取达到maxBytesInFlight大小的数据
 2. Iterator自动触发的next()方法，获取数据， 每一个循环里调用一次fetchUpToMaxBytes()获取一批数据
 3. 没有数据后，停止获取数据。
 
获取数据的API调用链如下：
```java
fetchUpToMaxBytes() -> fetchUpToMaxBytes()#send() -> sendRequest() -> shuffleClient.fetchBlocks()
```
后面的实现逻辑同上述`fetchBlocks()`获取数据的实现，同时不过多赘述获取数据的控制逻辑。

