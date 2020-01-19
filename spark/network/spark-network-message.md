# spark-network 消息模型

在网络传输通信中，不管我们传输的是序列化后的对象，还是文件，都是使用的字节流.
 * 在传统的IO模型中，字节流表现为Stream，比如`InputFileStream`.
 * 在NIO中，字节流表示为ByteBuffer；
 * 在Netty中字节流表示为ByteBuff或FileRegion；
 
在Spark中，通过封装ManagedBuffer 类，针对Byte也做了一层包装，支持对Byte和文件流进行处理和相互转换， 是network 框架的通信消息基础类。


## ManagedBuffer

ManagerBuffer 接口实现以下方法：


  * 获取该消息的bytes的大小， 如果消息需要被解密成其他形式的数据，则是解密后的数据大小。
    
    ```public abstract long size();  ```
  * 将数据转换成NIO支持的格式ByteBuffer，但是该操作需要消耗较大量的内存去做数据mapping， 不建议使用.
    
    ```public abstract ByteBuffer nioByteBuffer() throws IOException; ```
  * 将当前的buffer的数据转成InputStream 字节流，继承实现的方法不会去检查数据的长度，所以需要注意不会超出内存限制，比如继承类FileSegmentManagedBuffer,将文件转成字节流，需要注意文件的大小
    
    ```public abstract InputStream createInputStream() throws IOException;```
  * 如果试用，引用的计数+1

    ```public abstract ManagedBuffer retain();```
  * 如果是适用的，引用计数-1， 如果当引用计数为0，则释放内存
  
     ```public abstract ManagedBuffer release();```
  * 把当前的MessageBuffer 转换成Netty可使用的对象， ByteBuf/FileRegion {@link io.netty.buffer.ByteBuf} or a {@link io.netty.channel.FileRegion}.
    如果返回的是ByteBuf，相应的MessageBuffer的引用计数+1，由caller 需要去处理释放的过程
    
    ```public abstract Object convertToNetty() throws IOException;```


ManagedBuffer 的三个继承类，分别实现对应的转换方法， 实现了NIO Netty 和 文件字节流之间的灵活信息转换。

### FileSegmentManagedBuffer
表示一个文件的文件片段，内部变量存储文件的起始位position和要读取的文件长度。
在`createInputStream()`方法中，会生成length长度的LimitedInputStream。
在生成NIO传输对象`nioByteBuffer()`时，当length的长度小与设置的允许内存map的大小时，
调用`ByteBuffer.allocate()`方法，申请指定长度的DirectMemory 内存，直接copy文件的字节流，
读取对一个length长度的文件字节流。 当length的长度大于允许设置的内存map大小的size时，
直接使用memory mapping去复制对应的数据。 

当length长度较小的时候不是哟过memory mapping的原因是，memory mapping的开销时比较大的，给比较小的数据块就去做
memory mapping， 额外的overhead 过大，不划算， 所以设置一个范围，不与许对值小于 `spark.storage.memoryMapThreshold` 
设置的值的数据做memory mappig。
**后续需要了解memory mapping**

### NioManagedBuffer
NioManagedBuffer 是NIO传递信息 ByteBuffer 的包装，继承实现ManagedBuffer的方法，

### NettyManagedBuffer
NettyManagedBuffer 是Netty通信数据 ByteBuf 继承 managedBuffer的封装。



## 消息类型关系
在Spark的网络通信框架中， 主要有以下的基础消息类型：

### ChunkFetchRequest
请求获取数据流中的一个序列块，会返回 ChunkFetchSuccess 或者 ChuckFetchFailure

### ChunkFetchSuccess
ChunkFetchRequest 成功返回的对象， 返回的数据块为ManagedBuffer

### ChunkFetchFailure
ChunkFetchRequest 失败的返回对象，携带请求失败的错误消息。

### RpcRequest
一个被RPCHandler 使用的通用RPC请求，只会有一个返回，RPC请求类携带消息请求题ManagedBuffer

### RpcResponse
一个成功的RPC请求的返回类

### RpcFailure
RPC请求失败的返回类，携带请求失败的错误消息。

## OneWayMessage
一个不需要返回的单向RPC请求， 所以其只携带消息体ManagedBuffer， 而没有请求ID

## StreamRequest
向远端请求返回流数据的请求，对应请求的StreamId是在请求钱双端协商好的一个任意字符串

## StreamResponse
StreamRequest成功时的响应类。

需要注意，Responds消息本身不包含流数据。它是由发送者单独编写的。接收方需要设置一个临时的通道处理程序，该处理程序将消耗消息所表示的流的字节数。
实际的stream的消息的发送和接收， 是通过StreamManager 处理的， 可见  ·TransportRequestHandler· 的 `processStreamRequest()` 方法。

### StreamFailure
流请求失败的返回类，携带失败原因消息。

### UploadStream
带有能发送框架外部数据的RPC请求，因此可以将数据当作流来读取。





