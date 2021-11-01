# 集群echache相关
+ 重要的是配置CacheManager
+ 在这个基础下面面向cache的接口都是一样的。
+ 配置CacheManager最重要的是配置FactoryConfiguration

## 组播策略
+ 通知失效策略
     + 即各节点独立维护自己的缓存，当一个节点的某缓存发生变化时，并不将改变化同步复制到其他节点。而是通知其他节点清除该缓存。

  
     
## Q & A
+ 直接引用[1]中的做法同步的时候有问题，在后面启动的节点不能够同步显现已经put去的缓存，只能同步大小。即后面启动的节点可能会存在取不到前面节点存储缓存的情况。
+ 怎么样确保节点启动的时候能够同步正确已经有的ehcache的内容
     

     
     
     
     
     
## 参考链接
+ 1:[Ehcache使用JGroups做组播集群 https://nail2008.github.io/2017/06/16/distributed-ehcache-with-jgroups/](https://nail2008.github.io/2017/06/16/distributed-ehcache-with-jgroups/)