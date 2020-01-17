# spark-network message

在网络传输通信中，不管我们传输的是序列化后的对象，还是文件，都是使用的字节流.
在传统的IO模型中，字节流表现为Stream，比如`InputFileStream`.
在NIO中，字节流表示为ByteBuffer；
在Netty中字节流表示为ByteBuff或FileRegion；
在Spark中，针对Byte也做了一层包装，支持对Byte和文件流进行处理，即ManagedBuffer. network 框架的通信消息基础类。


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
  * 把当前的MessageBuffer 转换成Netty可食用的对象， ByteBuf/FileRegion {@link io.netty.buffer.ByteBuf} or a {@link io.netty.channel.FileRegion}.
    如果返回的是ByteBuf，相应的MessageBuffer的引用计数+1，由caller 需要去处理释放的过程
    
    ```public abstract Object convertToNetty() throws IOException;```


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

### 






