# FD_ALL2
基于简单心跳协议的故障检测。 每个成员定期（间隔ms）组播心跳。  每个成员还维护一个包含所有成员的表（减去其自身）。 当收到来自P的数据或心跳时，将与P关联的标志设置为true。 会定期检查过期成员，并怀疑那些标志为false的人（在超时毫秒内没有收到心跳或消息）。

## 参数
字段 | 默认值 | 含义
--- | --- | ---
interval |  8000 | 给集群发送心跳间隔
timeout | 40000 |超时之后，如果既没有从P接收到心跳也没有数据，则怀疑节点P
msg_counts_as_heartbeat | false | 将从成员收到的消息视为心跳。请注意，这意味着每次消息通过FD_ALL2向堆栈传递时，我们都会更新散列映射中的值，这很昂贵。默认值为false


## 实现
### 心跳HeartbeatSender
```
class HeartbeatSender implements Runnable {
        public void run() {
            Message heartbeat = new Message().setFlag(Message.Flag.INTERNAL).putHeader(id, new HeartbeatHeader());
            down_prot.down(new Event(Event.MSG, heartbeat));
            num_heartbeats_sent++;
        }

        public String toString() {
            return FD_ALL2.class.getSimpleName() + ": " + getClass().getSimpleName();
        }
    }
```
可以看到只是进行消息的广播这里只需要注意的一点是
 Message heartbeat = new Message().setFlag(Message.Flag.INTERNAL).putHeader(id, new HeartbeatHeader());
 表示当前的消息是心跳消息
 当别的节点收到广播信息的处理。直接看
 up方法
 ```
 public Object up(Event evt) {
        switch (evt.getType()) {
            case Event.MSG:
                Message msg = (Message) evt.getArg();
                Address sender = msg.getSrc();

                Header hdr = msg.getHeader(this.id);
                if (hdr != null) {
                    update(sender); // updates the heartbeat entry for 'sender'
                    num_heartbeats_received++;
                    unsuspect(sender);
                    return null; // consume heartbeat message, do not pass to the layer above
                } else if (msg_counts_as_heartbeat) {
                    // message did not originate from FD_ALL2 layer, but still count as heartbeat
                    update(sender); // update when data is received too ? maybe a bit costly
                    if (has_suspected_mbrs)
                        unsuspect(sender);
                }
                break; // pass message to the layer above
        }
        return up_prot.up(evt); // pass up to the layer above us
    }
 ```
看到当Header hdr = msg.getHeader(this.id);
hdr 不为null的时候表示的是接收到的上面的心跳消息
那么就更新节点的心跳信息同时解除怀疑该对象。

这里还有一个扩展的看点是如果不是心跳消息但是设置了msg_counts_as_heartbeat 参数也进行同样的处理。即收到消息就当成别的节点是存活的。但是这正如上面参数介绍的，代价很大（具体多大需要测试数据支持...）

另一个关注点是该心跳线程什么时候调用，什么时候启动。

直接看startHeartbeatSender方法。

```
timer.scheduleWithFixedDelay(new HeartbeatSender(), 1000, interval, TimeUnit.MILLISECONDS);
```
可以看到每interval 毫秒内心跳一次。

startHeartbeatSender方法是在
```
 protected void handleViewChange(View v) {
        List<Address> mbrs = v.getMembers();

        synchronized (this) {
            members.clear();
            members.addAll(mbrs);
            if (suspected_mbrs.retainAll(mbrs))
                has_suspected_mbrs = !suspected_mbrs.isEmpty();
            timestamps.keySet().retainAll(mbrs);
        }

        for (Address member : mbrs)
            update(member);

        if (mbrs.size() > 1) {
            startHeartbeatSender();
            startTimeoutChecker();
        } else {
            stopHeartbeatSender();
            stopTimeoutChecker();
        }
    }
```

处理视图改变的事件中被启动。
当视图改变时，判断视图中如果有超过一个节点则启动心跳线程。
否则取消。
再看handleViewChange方法是在什么时候进行调用的。
看到是在down方法里面
```
public Object down(Event evt) {
        switch (evt.getType()) {
            case Event.VIEW_CHANGE:
                down_prot.down(evt);
                View v = (View) evt.getArg();
                handleViewChange(v);
                return null;
            case Event.SET_LOCAL_ADDRESS:
                local_addr = (Address) evt.getArg();
                break;
            case Event.UNSUSPECT:
                Address mbr = (Address) evt.getArg();
                unsuspect(mbr);
                update(mbr);
                break;
        }
        return down_prot.down(evt);
    }
```
从这里可以得出推论
1: 视图的改变是由FD_ALL2更上层协议引起的，同时视图的管理不是由FD_ALL2控制。
2: 只有当视图里面超过2个节点时才会进行心跳
3: 心跳是面向于同一个channel的

那么这就可能引起一个问题。
即
如果视图分裂了，a节点的视图里面只有自己，那么他自己永远不会进行心跳。那么什么时候视图才会改变呢？

先存疑再看更新节点状态的定时器
上面对handleViewChange的分析中看到同时调用了
startTimeoutChecker 这就是节点更新器的启动时机。
但是有一点不同的就是间隔时间
```
timeout_checker_future = timer.scheduleWithFixedDelay(new TimeoutChecker(), timeout, timeout, TimeUnit.MILLISECONDS);
```
在默认的设置中timeout为40000，interval为8000
timeout远大于interval这是为了误伤友军。

### 更新别的节点状态TimeoutChecker
```
class TimeoutChecker implements Runnable {

        public void run() {
            List<Address> suspects = new LinkedList<>();
            for (Iterator<Entry<Address, AtomicBoolean>> it = timestamps.entrySet().iterator(); it.hasNext(); ) {
                Entry<Address, AtomicBoolean> entry = it.next();
                Address key = entry.getKey();
                AtomicBoolean val = entry.getValue();
                if (val == null) {
                    it.remove();
                    continue;
                }
                if (!val.compareAndSet(true, false)) {
                    log.debug("%s: haven't received a heartbeat from %s in timeout period (%d ms), adding it to suspect list",
                            local_addr, key, timeout);
                    suspects.add(key);
                }
            }
            if (!suspects.isEmpty())
                suspect(suspects);
        }

        public String toString() {
            return FD_ALL2.class.getSimpleName() + ": " + getClass().getSimpleName() + " (timeout=" + timeout + " ms)";
        }
    }
```
1：从timestamps列表里面拿取每个节点的心跳信息
如果里面的值为null 则直接从怀疑列表里面去掉。
2：读取AtomicBoolean的值，如果为false则放入怀疑列表同时置为false

那么这边重要的问题就是。
Address对应的AtomicBoolean状态什么时候进行改变。
这里有一个状态就是
检查的时候置为false。
回到上面的up方法上面，如果接受到别的节点的心跳表信息则调用update方法。
```
protected void update(Address sender) {
        if (sender != null && !sender.equals(local_addr)) {
            AtomicBoolean heartbeat_received = timestamps.get(sender);
            if (heartbeat_received != null)
                heartbeat_received.compareAndSet(false, true);
            else
                timestamps.putIfAbsent(sender, new AtomicBoolean(true));
        }
    }
```
这里直接置为true。

因此整体的流程就是
1：收到节点心跳信息 -> 设置为true
2：节点检查的时候 判断是否为ture 如果不是加入怀疑列表。同时置为false。

这里重要的就是加入了怀疑列表之后的处理。直接看suspect方法。
```
protected void suspect(List<Address> suspects) {
        if (suspects == null || suspects.isEmpty())
            return;

        num_suspect_events += suspects.size();

        final List<Address> eligible_mbrs = new ArrayList<>();
        synchronized (this) {
            for (Address suspect : suspects) {
                suspect_history.add(new Tuple<>(suspect, System.currentTimeMillis()));
                suspected_mbrs.add(suspect);
            }
            eligible_mbrs.addAll(members);
            eligible_mbrs.removeAll(suspected_mbrs);
            has_suspected_mbrs = !suspected_mbrs.isEmpty();
        }

        // Check if we're coord, then send up the stack
        if (local_addr != null && !eligible_mbrs.isEmpty()) {
            Address first = eligible_mbrs.get(0);
            if (local_addr.equals(first)) {
                if (log.isDebugEnabled())
                    log.debug("suspecting " + getSuspectedMembers());
                for (Address suspect : suspects) {
                    up_prot.up(new Event(Event.SUSPECT, suspect));
                    down_prot.down(new Event(Event.SUSPECT, suspect));
                }
            }
        }
    }
```
可以看到只是往上，往下协议栈发送Event.SUSPECT 消息。

心跳和定时检查的逻辑是清楚了大概就是：

1：视图改变的时候如果节点列表里面的节点大于1则启动心跳和定时检查线程
2：收到别的节点心跳声之后就更新节点的状态，表示它是存活的，同时发送Event.UNSUSPECT消息，这是在unsuspect方法里面实现的。
3：定时检查脚本会检查节点状态列表中在指定时间内没有收到心跳的节点向上，向下协议栈广播怀疑事件。

但是还是引出了下面的问题。

## Q & A
+ 怀疑之后怎么反馈到分裂，或者说怎么样的怀疑才会影响到成员列表的改变。