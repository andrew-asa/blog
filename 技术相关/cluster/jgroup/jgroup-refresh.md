节点连接
问题的起源是REPORT-16248
排查过程：
问题的原因是节点启动的时候在ClusterTicket里面根据ClusterBridge.getView().getNodeById(xxx)
根据已经连接上集群的id拿不到对应信息。
本地代码调试，搭建centos web集群的时候也没有能够还原。
注意到测试采用的是window下的web 集群

然后翻看代码查看大概有那么一段
```
jChannel.setReceiver(new ReceiverAdapter() {

            @Override
            public void viewAccepted(View new_view) {
                EventDispatcher.fire(ClusterCommand.REFRESH, new_view);
            }
        });
        jChannel.connect(ProtocolStackType.CORE.getName());
```
同时知道
1：所有的ClusterTicket是在jChannel.connect之后。
2：ClusterBridge.getView().getNodeById(xxx) 拿不到说明还没有触发ClusterCommand.REFRESH事件或者发生了ClusterCommand.REFRESH事件但是在刷新的视图里面不存在已经

所以产生了第一个假设viewAccepted是异步事件，所以ClusterTicket里面能不能拿到整个完整视图完全是一个概论事件。本机和centos上搭建的集群能用，说明刚好发生在viewAccepted之后，假设到了这里就感到有点发麻了，因为这样的一个裂痕就有可能产生大问题。




节点连接流程

事情的发展流程
1: jChannel.connect(cluster_name);
2: 向jgroup协议栈下面发送Event.CONNECT_USE_FLUSH 事件
3: GMS的down事件的case Event.CONNECT_USE_FLUSH 分支进行处理
4: 调用GmsImpl.join 然后调用GmsImpl.joinInternal方法
5:直接上代码解析,省略了部分不是很重要的代码

```
protected void joinInternal(Address mbr, boolean joinWithStateTransfer, boolean useFlushIfPresent) {
        
        long join_attempts = 0;
        leaving = false;
        join_promise.reset();
        while (!leaving) {
            if (installViewIfValidJoinRsp(join_promise, false))
                return;
            
            long start = System.currentTimeMillis();
            Responses responses = (Responses) gms.getDownProtocol().down(Event.FIND_INITIAL_MBRS_EVT);
            
            if (installViewIfValidJoinRsp(join_promise, false))
                return;
      responses.waitFor(gms.join_timeout);
            responses.done();
            long diff = System.currentTimeMillis() - start;
            if (responses.isEmpty()) {
            becomeSingletonMember(mbr);
                return;
            }            
            List<Address> coords = getCoords(responses);
            if (coords == null) { // e.g. because we have all clients only
                if (firstOfAllClients(mbr, responses))
                    return;
                continue;
            }
            
            if (coords.size() > 1)            
            join_attempts++;
            for (Address coord : coords) {
                    sendJoinMessage(coord, mbr, joinWithStateTransfer, useFlushIfPresent);
                if (installViewIfValidJoinRsp(join_promise, true))
                    return;            
            if (gms.max_join_attempts != 0 && join_attempts >= gms.max_join_attempts) {
               becomeSingletonMember(mbr);
                return;
            }
        }
    }
```
+ 5.1 这里有几个经常出现的方法



# 节点刷新事件
+ JChannel.connect
+ 初始化套接字
+ ClientGmsImpl.join

## 相关问题
+ 如何获取到组里面的成员信息
+ 获取到组里面成员信[]()息之后是怎么进行处理
+ jgroup.connect连上集群组是同步事件还是异步事件？即viewChange事件是怎么进行的
+ 为什么在window 集群下面会出现两个viewChange事件？

## GMS
### 参数
+ join_timeout 加入 一组网络等待相应时间。如果第一次连接等待join_timeout时间还收不到相应的查找成员回应，则认为自己是主节点。

### 请求头
+ GmsHeader.JOIN_REQ    
    + 申请加入的节点已经获取到地址？
+ GmsHeader.JOIN_RSP 
    + jgroup connect的时候加入请求。 
+ GmsHeader.LEAVE_REQ         
+ GmsHeader.LEAVE_RSP      
+ GmsHeader.VIEW      
+ GmsHeader.MERGE_REQ 
+ GmsHeader.MERGE_RSP    
+ GmsHeader.INSTALL_MERGE_VIEW   
+ GmsHeader.CANCEL_MERGE   
+ GmsHeader.VIEW_ACK 
+ GmsHeader.JOIN_REQ_WITH_STATE_TRANSFER 
+ GmsHeader.INSTALL_MERGE_VIEW_OK 
+ GmsHeader.GET_DIGEST_REQ 
+ GmsHeader.GET_DIGEST_RSP 
+ GmsHeader.INSTALL_DIGEST 
+ GmsHeader.GET_CURRENT_VIEW 


## 参考链接
+ [https://xbuba.com/questions/54478034](https://xbuba.com/questions/54478034)