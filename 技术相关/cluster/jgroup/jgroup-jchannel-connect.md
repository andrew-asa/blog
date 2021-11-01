# jgroup 的连接管道过程

JChannel.connect 连接到集群通道
最终会调用
JChannel.down(Event.CONNECT_USE_FLUSH)
最终会调用协议栈的down方法，对这个事件感兴趣的是GMS协议
所以看
GMS.down()
直接调用
ClientGmsImpl.joinInternal方法


