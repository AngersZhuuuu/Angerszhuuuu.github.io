 
 # External Shuffle Service
 
Spark系统在运行含shuffle过程的应用时，Executor进程除了运行task，还要负责写shuffle 数据，给其他Executor提供shuffle数据。
当Executor进程任务过重，导致GC而不能为其 他Executor提供shuffle数据时，会影响任务运行。
这里实际上是利用External Shuffle Service 来提升性能，External shuffle Service是长期存在于NodeManager进程中的一个辅助服务。
通过该服务 来抓取shuffle数据，减少了Executor的压力，在Executor GC的时候也不会影响其他 Executor的任务运行。

External Shuffle Service的类：

 * ExternalShuffleService  服务与Standalone 模式的Worker上
 * YarnShuffleService  服务于Yarn的NodeManager上


以Yarn模式为例，介绍External Shuffle Service怎么交换数据

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
ExternalShuffleService中的BlockStoreClient继承类为 ExternalBlockStoreClient, fetchBlocks()方法如下

 @Override
  public void fetchBlocks(
      String host,
      int port,
      String execId,
      String[] blockIds,
      BlockFetchingListener listener,
      DownloadFileManager downloadFileManager) {
    checkInit();
    logger.debug("External shuffle fetch from {}:{} (executor id {})", host, port, execId);
    try {
      RetryingBlockFetcher.BlockFetchStarter blockFetchStarter =
          (blockIds1, listener1) -> {
            // Unless this client is closed.
            if (clientFactory != null) {
              TransportClient client = clientFactory.createClient(host, port);
              new OneForOneBlockFetcher(client, appId, execId,
                blockIds1, listener1, conf, downloadFileManager).start();
            } else {
              logger.info("This clientFactory was closed. Skipping further block fetch retries.");
            }
          };

      int maxRetries = conf.maxIORetries();
      if (maxRetries > 0) {
        // Note this Fetcher will correctly handle maxRetries == 0; we avoid it just in case there's
        // a bug in this code. We should remove the if statement once we're sure of the stability.
        new RetryingBlockFetcher(conf, blockFetchStarter, blockIds, listener).start();
      } else {
        blockFetchStarter.createAndStart(blockIds, listener);
      }
    } catch (Exception e) {
      logger.error("Exception while beginning fetchBlocks", e);
      for (String blockId : blockIds) {
        listener.onBlockFetchFailure(blockId, e);
      }
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