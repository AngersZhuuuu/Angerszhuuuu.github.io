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
