## redis 相关

### 笔记
+ 到[https://redis.io](https://redis.io)下载源码进行编译，然后输入`src/redis-server`即可以启动redis服务器
+ hello world 
   + ./redis-service
   + ./redis-cli
      + set hello world
      + get hello 
   
+ redis 连接远程客户端
    + 连接`redis-cli -h {redis_host} -p {redis_port}`
    + 停止，退出

+ redis service和client不处于同一个ip？

+ 对于java客户端来就，redis就应该当成是一种基本数据结构就可以，然后再提供全局查找的功能
+ redis-check-aof/redis-check-rdb 数据库恢复工具
+ redis-server 服务器程序
   + --dir db存放路径
   + --port 端口
+ redis-cli 客户端程序
+ redis-benchmark 性能测试程序



### Q & A
+ redis 只是用于状态管理器？
+ redis 如何保障多态服务器的访问的锁问题？
+ 配置Redis不要开放0.0.0.0（要不就配置密码），否则服务器会存在安全风险高?
   + A:[3]
+ redis 文件存放在哪里？
+ 怎么配置客户端连接配置密码?
+ 客户端连接是否有账户密码这一说法？
+ redis 单机于redis集群有什么关联以及区别？
+ 过个web 集群节点之间使用redis有什么需要注意的东西
+ 既然redis是一个内存key-value数据库，那么还需要使用者自己担心内存的问题，还是可以进行配置？
+ 说redis是内存操作似乎是不准确的，因为用redis-cli和服务器进行交互的时候关掉再重启还是可以获取之前保存的值？
+ 是否有字符集配置的说法？默认的字符集是什么？


### 参考资料
+ 1:[redis帮助文档 http://www.redis.cn/community.html](http://www.redis.cn/community.html)
+ 2:[redis在线命令学习 http://try.redis.io/](http://try.redis.io/)
+ 3:[Redis 未授权访问漏洞利用总结 http://www.alloyteam.com/2017/07/12910/](http://www.alloyteam.com/2017/07/12910/)