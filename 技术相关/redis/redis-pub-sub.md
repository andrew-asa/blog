# redis 发布和订阅

## 相关命令。
PSUBSCRIBE pattern [pattern ...] 
1 订阅一个或多个符合给定模式的频道。
2	PUBSUB subcommand [argument [argument ...]] 
查看订阅与发布系统状态。
3	PUBLISH channel message 
将信息发送到指定的频道。
4	PUNSUBSCRIBE [pattern [pattern ...]] 
退订所有给定模式的频道。
5	SUBSCRIBE channel [channel ...] 
订阅给定的一个或多个频道的信息。
6	UNSUBSCRIBE [channel [channel ...]] 
指退订给定的频道。

## jedis 发布和订阅操作。
+ 订阅

jedis. subscribe(final JedisPubSub jedisPubSub, final String... channels)
注意：
1： 这里的JedisPubSub 是需要自己实现一个，并实现其自己关注的onMessage(String channel, String message)等方法。
2：这里的subscribe是一个阻塞方法，所以设置完成并不会返回，所以需要把改方法放到线程里面进行
3：jedis,clusterJedis并没有提供有相应的退订放，需要退订，同时再次添加订阅频道可以直接操作
自定义的JedisPubSub对象。而不是再通过jedis对象进行操作。

+ 发布
jedis.publish(String channel, String message)
直接往相应的频道发布指定的消息即可。

[Jedis实现Publish/Subscribe功能](https://blog.csdn.net/lihao21/article/details/48370687)