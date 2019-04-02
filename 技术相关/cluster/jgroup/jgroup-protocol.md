# JGroup 相关协议

## UDP
+ 配置

## TCP
+ start_port 本节点监听的端口，默认为7800，如设置为0,将会随机选定一个端口绑定
+ end_port 尝试监听的最大端口，如果在start_port和end_port指定的范围内都找不到一个可用端口，则抛出一个BindException。如果不指定end_port，或者end_port<start_port，end_port将没有上限，如果start_port=end_port，JGroup将会只用该端口来绑定。
+ bind_addr 192.168.1.3" JGroup监听的物理IP，如不指定，JGroup将会绑定到所有可用网络接口。当有多个网络接口时应该指定一个

## UDP
+ mcast_addr 多播地址
+ mcast_port 多播端口


## MERGE3
+ 1：成员发现，成员更新协议
+ 2：更新的情况有多种
    + 2.1:view 更新
    + 2.2：接收消息的时候发现没有在成员列表里面的ip
    + 2.3：合并子分组
+ 3：探寻子组是否存在


+ down
   + down 下来如果是Event.VIEW_CHANGE事件
   则会检查视图是否一致
   
   
   ## 参考资料
   + [https://developer.jboss.org/wiki/JGroups](https://developer.jboss.org/wiki/JGroups)
      

