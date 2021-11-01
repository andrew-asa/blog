# 事件相关
+ Event.GET_PHYSICAL_ADDRESS 获取物理地址
+ Event.CONNECT 节点连接上集群
+ Event.CONNECT
+ Event.CONNECT_WITH_STATE_TRANSFER
+ Event.CONNECT_USE_FLUSH
+  Event.CONNECT_WITH_STATE_TRANSFER_USE_FLUSH

+ Event.FIND_INITIAL_MBRS_EVT 获取组成员信息

事件| 值 | 入参 | 出参 | 处理类 | 作用
--- | ---| ---| --- | --- | ---
MSG | 1 | Message | | | 信息发送类
FIND_INITIAL_MBRS | 12 | null | Responses | Discovery | 查询组内成员信息
LOCK | 95 | LockInfo | | | 获取锁
UNLOCK | 96 | LockInfo | | | 解锁