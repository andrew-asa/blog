## FD_SOCK 
故障检测协议基于在集群成员之间创建的TCP套接字环，类似于FD但不使用心跳消息。

+ 基于2个成员之间的TCP连接的故障检测
+ 在视图{A，B，C}中，A连接到B，B连接到C，C连接回A
+ 基于组成员之间创建的TCP套接字环的故障检测协议。组中的每个成员连接到其邻居（最后一个成员连接到第一个成员），从而形成一个环。
+ 当邻居A检测到异常关闭的TCP套接字（可能是由于节点B崩溃）时怀疑成员B。但是如果一名成员B即将优雅地离开，它会让其邻居A知道，这样就不会被怀疑。
+ 一个FD_SOCK的缺点是挂起的服务器和/或崩溃的交换机不会导致套接字被关闭。
+ 因此，不会怀疑挂起的成员，并且不会检测到由于交换机故障导致的网络分区。

+ FD和FD_SOCK之间的区别
    + FD
        + 重启节点在发送活动响应时可能会很慢
        + 虚假怀疑
        + 在debugger/profiler中暂停节点将被怀疑
        + 低超时导致错误怀疑的可能性更高
        + 高超时一段时间内不会检测并删除崩溃的成员。
        
    + FD_SOCK
        + 暂停在调试器中没问题
        + 高负荷也没问题
        + 仅在TCP连接中断时才会怀疑成员

        

## 参数
字段 | 默认值 | 含义
--- | --- | ---
bind_addr |   | ServerSocket应侦听的NIC。还会识别以下特殊值：GLOBAL, SITE_LOCAL, LINK_LOCAL and NON_LOOPBACK
cache_max_age |   | 标记为已删除的元素的最大年龄（以毫秒为单位）必须具有删除元素
cache_max_elements |   | 删除删除元素之前缓存中的最大元素数
client_bind_port |   | 启动客户端套接字的端口。默认值为0随机端口选择
external_addr |   | 如果您在防火墙后面的不同网络上拥有主机，请使用“external_addr”。在每个防火墙上，设置一个端口转发规则（有时称为“虚拟服务器”）到主机的本地IP（例如192.168.1.100），然后在每个主机上，将“external_addr”TCP传输参数设置为防火墙的外部（公共IP）地址。
external_port |   | 用于将内部端口（bind_port）映射到外部端口。仅在> 0时使用
get_cache_timeout |   | 从协调器获取套接字缓存的超时
keep_alive |   | 是否在ping套接字上使用KEEP_ALIVE。默认为truenum_tries |   | 
num_tries |   | 放弃之前，请求协调器请求套接字缓存的次数
port_range |   | 探测start_port和client_bind_port的端口数
sock_conn_timeout |   | 以毫秒为单位的最大时间等待ping Socket.connect（）返回
start_port |   | 启动服务器套接字端口默认值为0随机端口选择
suspect_msg_interval |   | 广播可疑消息的间隔

