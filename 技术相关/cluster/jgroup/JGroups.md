# JGroups 
+ 群组通讯工具包
+ 集群的功能从大的方向来说就是一个群组
+ 集群只是一个概念，任何能够协调多台机器同时工作的的模块都叫做集群


## 配置
```
<!--
    TCP based stack, with flow control and message bundling. This is usually used when IP
    multicasting cannot be used in a network, e.g. because it is disabled (routers discard multicast).
    Note that TCP.bind_addr and TCPPING.initial_hosts should be set, possibly via system properties, e.g.
    -Djgroups.bind_addr=192.168.5.2 and -Djgroups.tcpping.initial_hosts=192.168.5.2[7800]
    author: Bela Ban
-->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns="urn:org:jgroups"
        xsi:schemaLocation="urn:org:jgroups http://www.jgroups.org/schema/jgroups.xsd">
    <TCP bind_port="${START_PORT}"
         port_range="6"
         recv_buf_size="${tcp.recv_buf_size:5M}"
         send_buf_size="${tcp.send_buf_size:5M}"
         max_bundle_size="64K"
         max_bundle_timeout="30"
         use_send_queues="true"
         sock_conn_timeout="300"
         bind_addr="${BIND_ADDR}"

         timer_type="new3"
         timer.min_threads="4"
         timer.max_threads="10"
         timer.keep_alive_time="3000"
         timer.queue_max_size="500"

         thread_pool.enabled="true"
         thread_pool.min_threads="10"
         thread_pool.max_threads="20"
         thread_pool.keep_alive_time="5000"
         thread_pool.queue_enabled="true"
         thread_pool.queue_max_size="5000"
         thread_pool.rejection_policy="discard"

         oob_thread_pool.enabled="true"
         oob_thread_pool.min_threads="4"
         oob_thread_pool.max_threads="8"
         oob_thread_pool.keep_alive_time="5000"
         oob_thread_pool.queue_enabled="false"
         oob_thread_pool.queue_max_size="100"
         oob_thread_pool.rejection_policy="discard"/>

    <TCPPING async_discovery="true"
             initial_hosts="${INITIAL_HOSTS}"
             port_range="6"/>
    <MERGE3 min_interval="10000"
            max_interval="30000"/>
    <FD_SOCK/>
    <FD_ALL2 interval="3000" timeout="15000"/>
    <VERIFY_SUSPECT timeout="1500"/>
    <BARRIER/>
    <pbcast.NAKACK2 use_mcast_xmit="false"
                    discard_delivered_msgs="true"/>
    <UNICAST3/>
    <pbcast.STABLE stability_delay="1000" desired_avg_gossip="50000"
                   max_bytes="4M"/>
    <pbcast.GMS print_local_addr="true" join_timeout="5000"
                view_bundling="true"/>
    <MFC max_credits="2M"
         min_threshold="0.4"/>
    <FRAG2 frag_size="60K"/>
    <!--RSVP resend_interval="2000" timeout="10000"/-->
    <pbcast.STATE_TRANSFER/>
    <CENTRAL_LOCK num_backups="2"/>
</config>
```

```
CENTRAL_LOCK
pbcast.STATE_TRANSFER
FRAG2
MFC
pbcast.GMS
pbcast.STABLE
UNICAST3
pbcast.NAKACK2
BARRIER
VERIFY_SUSPECT
FD_ALL2
FD_SOCK
MERGE3
TCPPING
TCP
```

# Q & A
+ EhCache 从 1.5. 版本开始增加了 JGroups 的分布式集群模式。与 RMI 方式相比较， JGroups 提供了一个非常灵活的协议栈、可靠的单播和多播消息传输，主要的缺点是配置复杂以及一些协议栈对第三方包的依赖。 
  + 什么是单播，什么是多播？主要是用于解决什么问题?
+ 基于JGroup 就可以搭建一个简单的集群环境？怎么来？
+ 组里面的节点范围可以多广？ip分布可以多广？广播的时候又是如何进行标明，加入某一个组？
+ 成员发现，协调者发现机制是怎么样的？
+ 加入通道的操作消息传播范围有多广？
+ 节点命名？

## 参考资料
+ [JGroup 手册 http://jgroups.org/manual/index.html](http://jgroups.org/manual/index.html)
+ [https://www.ibm.com/developerworks/cn/java/j-lo-ehcache/index.html](https://www.ibm.com/developerworks/cn/java/j-lo-ehcache/index.html)
+  [http://www.itboth.com/d/3mMbua/socket-tcp-javadoc-jms](http://www.itboth.com/d/3mMbua/socket-tcp-javadoc-jms)
+  [Ehcache 下 用jgroup进行复制，Ehcache集群 https://blog.csdn.net/kindy1022/article/details/6681299](https://blog.csdn.net/kindy1022/article/details/6681299)
+  [使用JGroups TCP实现EHCache的集群 https://my.oschina.net/u/866380/blog/501082](https://my.oschina.net/u/866380/blog/501082)
+  [jgroup 官方文档 http://www.jgroups.org/](http://www.jgroups.org/)
+  [JGroup 介绍 https://gcloud.qq.com/forum/topic/56a58c5188d0a30a0ddb176f](https://gcloud.qq.com/forum/topic/56a58c5188d0a30a0ddb176f)
+  [JGroup 相关功能 https://developer.jboss.org/en/jgroups/content?itemView=thumbnail&_sscc=t](https://developer.jboss.org/en/jgroups/content?itemView=thumbnail&_sscc=t)
+  [https://whitesock.iteye.com/blog/199229](https://whitesock.iteye.com/blog/199229)
+  [协议栈讲解 https://blog.csdn.net/xianymo/article/details/44064547](https://blog.csdn.net/xianymo/article/details/44064547)
+  [JGroups入门教程BARRIER层  https://www.linuxidc.com/Linux/2013-10/91114p5.htm](https://www.linuxidc.com/Linux/2013-10/91114p5.htm)
+  [Reliable group communication with JGroups](https://blog.csdn.net/tengdazhang770960436/article/details/49472169)
+  [http://www.jgroups.org/manual4/index.html#FD_SOCK](http://www.jgroups.org/manual4/index.html#FD_SOCK)
+  []()