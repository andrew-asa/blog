### 源码解析
## 涉及到的类
## 事件
+ connect
    + 前台代码
```
var socket =  io.connect('http://localhost:9092');
```
### 遇到的问题
+ 同一个浏览器不同的标签页带的sessionid一致
+ https://stackoverrun.com/cn/q/3427996

### 参考资源
+ 1:[https://github.com/mrniko/netty-socketio/tree/netty-socketio-1.7.16 
](https://github.com/mrniko/netty-socketio/tree/netty-socketio-1.7.16 
)
+ 2: [https://socket.io/docs/client-api/](https://socket.io/docs/client-api/)
+ 3: [https://zhuanlan.zhihu.com/p/29148869](https://zhuanlan.zhihu.com/p/29148869)
+ 4: [https://www.jianshu.com/p/a3e06ec1a3a0](https://www.jianshu.com/p/a3e06ec1a3a0)