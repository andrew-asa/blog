# RPC 相关

## 零散笔记
+ RPC需要解决的问题： 
  + 通讯问题 : 主要是通过在客户端和服务器之间建立TCP连接，远程过程调用的所有交换的数据都在这个连接里传输。连接可以是按需连接，调用结束后就断掉，也可以是长连接，多个远程过程调用共享同一个连接。
  + 寻址问题 ： A服务器上的应用怎么告诉底层的RPC框架，如何连接到B服务器（如主机或IP地址）以及特定的端口，方法的名称名称是什么，这样才能完成调用。比如基于Web服务协议栈的RPC，就要提供一个endpoint URI，或者是从UDDI服务上查找。如果是RMI调用的话，还需要一个RMI Registry来注册服务的地址。
 + 序列化 与 反序列化 ： 当A服务器上的应用发起远程过程调用时，方法的参数需要通过底层的网络协议如TCP传递到B服务器，由于网络协议是基于二进制的，内存中的参数的值要序列化成二进制的形式，也就是序列化（Serialize）或编组（marshal），通过寻址和传输将序列化的二进制发送给B服务器。 
同理，B服务器接收参数要将参数反序列化。B服务器应用调用自己的方法处理后返回的结果也要序列化给A服务器，A服务器接收也要经过反序列化的过程。



## 参考资料
+ [深入理解RPC之动态代理篇 https://www.cnkirito.moe/rpc-dynamic-proxy/](https://www.cnkirito.moe/rpc-dynamic-proxy/)
+ [RPC框架几行代码就够了 https://javatar.iteye.com/blog/1123915](https://javatar.iteye.com/blog/1123915)
+ [https://blog.csdn.net/loongshawn/article/details/73903709](https://blog.csdn.net/loongshawn/article/details/73903709)