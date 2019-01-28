## redis 相关

### 零散知识点
+ 到[https://redis.io](https://redis.io)下载源码进行编译，然后输入`src/redis-server`即可以启动redis服务器
+ hello world 
   + ./redis-service
   + ./redis-cli
      + set hello world
      + get hello 

+ redis service和client不处于同一个ip？

+ 对于java客户端来就，redis就应该当成是一种基本数据结构就可以，然后再提供全局查找的功能
+ 


### Q & A
+ redis 只是用于状态管理器？
+ redis 如何保障多态服务器的访问的锁问题？
+ 配置Redis不要开放0.0.0.0（要不就配置密码），否则服务器会存在安全风险高?
+ redis 文件存放在哪里？
+ 怎么配置客户端连接配置密码?
+ 客户端连接是否有账户密码这一说法？


### 参考资料
+ 1:[redis帮助文档 http://www.redis.cn/community.html](http://www.redis.cn/community.html)
+ 2:[redis在线命令学习 http://try.redis.io/](http://try.redis.io/)