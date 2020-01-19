# spark-network 的消息发送和接收处理框架

## 发送
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
 
 