# redis 集群相关

## 执行脚本

## Q & A
+ 集群的每个节点是否并不会保存所有节点的key-value对？
+ 主从结构的时候，从节点什么时候回从主节点里面进行key-value对复制，是否是实时的？
    + 是异步的。
+ 主节点down掉之后，从节点是否立马就能进行处理之前主节点的key-value请求，其它节点又是否能里面应该把转发给之前down掉的主节点的key-value请求正确的转发给从节点。
+ 如果一个主节点和一个从节点都down掉之后，整个redis集群是否还是可用？


## 参考连接
+ [Redis集群快速启动脚本程序 https://blog.csdn.net/fengyong7723131/article/details/53196382](https://blog.csdn.net/fengyong7723131/article/details/53196382)
+ [https://blog.csdn.net/moxiaomomo/article/details/17540813](https://blog.csdn.net/moxiaomomo/article/details/17540813)
[Redis Cluster 分区实现原理 http://blog.jobbole.com/103258/](http://blog.jobbole.com/103258/)
+ [redis主从宕机数据恢复问题 https://wuyanteng.github.io/2017/11/23/redis%E4%B8%BB%E4%BB%8E%E5%AE%95%E6%9C%BA%E6%95%B0%E6%8D%AE%E6%81%A2%E5%A4%8D%E9%97%AE%E9%A2%98/](https://wuyanteng.github.io/2017/11/23/redis%E4%B8%BB%E4%BB%8E%E5%AE%95%E6%9C%BA%E6%95%B0%E6%8D%AE%E6%81%A2%E5%A4%8D%E9%97%AE%E9%A2%98/)
+ [Redis哨兵机制 https://segmentfault.com/a/1190000018278099](https://segmentfault.com/a/1190000018278099)
+ [美团在Redis上踩过的一些坑-5.redis cluster遇到的一些问题 https://carlosfu.iteye.com/blog/2254573](https://carlosfu.iteye.com/blog/2254573)
+ [https://blog.csdn.net/gqtcgq/article/details/51830428](https://blog.csdn.net/gqtcgq/article/details/51830428)
+ [https://redis.io/topics/cluster-spec](https://redis.io/topics/cluster-spec)