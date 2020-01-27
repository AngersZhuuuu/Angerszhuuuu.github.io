# spark-network server端的消息发送和接收处理框架

## server端接收
在Spark的network 框架中，消息的服务方是类 `TransportServer`，该类的对象由TransportContext 所构造，

该类在启动的时候启动 Netty的 `ServerBootstrap`,用于接收网络IO中的请求， 并由TransportServer的内部
RpcHandler appRpcHandler对象， 对channel 中的消息进行处理