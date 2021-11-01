# GMS
GMS即群组成员关系协议，该协议时JGroups协议栈中的重要协议，它维护者一个活着节点的列表。GMS负责群组成员加入和离开群组的请求，同时它也处理错误探测协议发送的SUSPECT协议。

## 算法
组成员资格负责加入新成员，处理现有成员的请假请求，以及处理由故障检测协议发出的崩溃成员的SUSPECT消息。加入新成员的算法基本上是：

- 循环
- 查找初始化成员（发现协议）
- 如果没有响应
    - 成为单组然后中断循环
- 否则
    - 决定协调者（响应中最老的成员）
    - 发送join请求到协调者
    - 等待join 请求的相应
    - 如果接收到join 请求相应
        - 安装视图然后中断循环
    - 否则
        - 休眠5s然后继续循环

## 参数
字段 | 默认值 | 含义
--- | --- | ---
install_view_locally_first | false | 是否在广播之前首先在本地安装新视图（仅在协调处完成）。如果检测到状态传输协议，则自动设置为true
join_timeout |  | 加入超时
leave_timeout |  | 离开超时
log_collect_msgs |  | 如果为true，则记录收集所有视图的失败
log_view_warnings |  | 记录接收低于当前视图的警告，以及不包括自己的视图
max_bundling_time |  | 如果打开视图捆绑，则最大视图捆绑超时
max_join_attempts |  | 在我们放弃并成为单身之前的连接尝试次数。 0意味着永不放弃。
membership_change_policy |  | 实现MembershipChangePolicy的类的完全限定名称。
merge_timeout |  | 视图合并超时（以毫秒为位）
num_prev_mbrs |  | 历史记录中保留的最大旧成员数。默认值为50
num_prev_views |  | 要存储在历史记录中的视图数
print_local_addr |  | 连接后打印该成员的本地地址。默认为true
print_physical_addrs |  | 在启动时打印物理地址
use_delta_views |  | 如果为true，则允许GMS发送带有delta视图的VIEW消息，否则它始终发送完整视图。
use_flush_if_present | true | 使用flush进行视图更改。默认为true
view_ack_collection_timeout | 2000 | 以ms为单位等待所有VIEW确认的时间（0 ==永远等待。默认为2000毫秒
view_bundling |  | 查看捆绑切换

## 加入成为一个成员
请考虑以下情况：新成员想要加入一个组。这样做的顺序是：
+ 组播（不可靠）发现请求（ping）
+ 等待n个响应或m毫秒（以先到者为准）
+ 每个成员都以协调员的地址作出回应
+ 如果初始响应> 0：确定协调器并启动JOIN协议
+ 如果初始响应为0：成为协调员，假设没有其他人在那里

然而，问题是初始mcast发现请求可能会丢失，例如，当多个成员同时启动时，传出网络缓冲区可能会溢出，并且mcast数据包可能会被丢弃。没有人收到它，因此发件人不会收到任何回复，导致初始成员资格为0这可能导致多个协调器和多个子组形成。我们怎样才能克服这个问题呢？有两种解决方案：
1：增加超时或收到的响应数量。如果成员资格空虚的原因是主机较慢，这只会有所帮助。如果丢弃mcast数据包，此解决方案将无济于事
2：添加MERGE2或者MERGE3协议，这实际上并不会阻止多个初始的协调者，而是通过将不同的子群合并为一个来纠正问题。请注意，这可能涉及需要由应用程序完成的状态合并。

## 源码分析
### 主线1：Request.SUSPECT
在FD_ALL2中知道一旦一个节点在timeout时间内收不到节点的心跳就会向VERIFY_SUSPCET协议发送一个Request.SUSPECT事件，当VERIFY_SUSPCET在timeout时间内收不到怀疑节点的VerifyHeader.ARE_YOU_DEAD回复就会向GMS
协议发送一个Request.SUSPECT消息。
所以，分析的起点是GMS的up方法

```
case Event.SUSPECT:
                Object retval = up_prot.up(evt);
                Address suspected = (Address) evt.getArg();
                view_handler.add(new Request(Request.SUSPECT, suspected, true));
                ack_collector.suspect(suspected);
                merge_ack_collector.suspect(suspected);
                return retval;
```
做的事情有
1：view_handler.add(new Request(Request.SUSPECT, suspected, true));
2：ack_collector.suspect(suspected);
3：merge_ack_collector.suspect(suspected);

先看1
这里的view_handler是GMS.ViewHandler对象
add 方法
```
synchronized void add(Request req) {
            if (suspended) {
                log.trace("%s: queue is suspended; request %s is discarded", local_addr, req);
                return;
            }
            start();
            try {
                queue.add(req);
                history.add(new Date() + ": " + req.toString());
            } catch (QueueClosedException e) {
                log.trace("%s: queue is closed; request %s is discarded", local_addr, req);
            }
        }

```
```
 synchronized void start() {
            if (queue.closed())
                queue.reset();
            if (thread == null || !thread.isAlive()) {
                thread = getThreadFactory().newThread(this, "ViewHandler");
                thread.setDaemon(false); // thread cannot terminate if we have tasks left, e.g. when we as coord leave
                thread.start();
            }
        }
```
直接往队列里面添加一个请求然后启动线程
常用套路直接看GMS.ViewHandler.run方法
```
public void run() {
            long start_time, wait_time;  // ns
            long timeout = TimeUnit.NANOSECONDS.convert(max_bundling_time, TimeUnit.MILLISECONDS);
            List<Request> requests = new LinkedList<>();
            while (Thread.currentThread().equals(thread) && !suspended) {
                try {
                    boolean keepGoing = false;
                    start_time = System.nanoTime();
                    do {
                        Request firstRequest = (Request) queue.remove(INTERVAL); // throws a TimeoutException if it runs into timeout
                        requests.add(firstRequest);
                        if (!view_bundling)
                            break;
                        if (queue.size() > 0) {
                            Request nextReq = (Request) queue.peek();
                            keepGoing = view_bundling && firstRequest.canBeProcessedTogether(nextReq);
                        } else {
                            wait_time = timeout - (System.nanoTime() - start_time);
                            if (wait_time > 0 && firstRequest.canBeProcessedTogether(firstRequest)) { // JGRP-1438
                                long wait_time_ms = TimeUnit.MILLISECONDS.convert(wait_time, TimeUnit.NANOSECONDS);
                                if (wait_time_ms > 0)
                                    queue.waitUntilClosed(wait_time_ms); // misnomer: waits until element has been added or q closed
                            }
                            keepGoing = queue.size() > 0 && firstRequest.canBeProcessedTogether((Request) queue.peek());
                        }
                    }
                    while (keepGoing && timeout - (System.nanoTime() - start_time) > 0);

                    try {
                        process(requests);
                    } finally {
                        requests.clear();
                    }
                } catch (QueueClosedException e) {
                    break;
                } catch (TimeoutException e) {
                    break;
                } catch (Throwable catchall) {
                    Util.sleep(50);
                }
            }
        }
```
只要关注process(requests);即可
直接看process方法
```
private void process(List<Request> requests) {
            if (requests.isEmpty())
                return;
            Request firstReq = requests.get(0);
            switch (firstReq.type) {
                case Request.JOIN:
                case Request.JOIN_WITH_STATE_TRANSFER:
                case Request.LEAVE:
                case Request.SUSPECT:
                    impl.handleMembershipChange(requests);
                    break;
                case Request.MERGE:
                    impl.merge(firstReq.views);
                    break;
                default:
                    log.error("request " + firstReq.type + " is unknown; discarded");
            }
        }
```
这里的impl为GmsImpl对象，
对于协调者来说指的是CoordGmsImpl
对于一般节点来说指的是ParticipantGmsImpl

先看一般节点的处理方式
ParticipantGmsImpl.handleMembershipChange
```
public void handleMembershipChange(Collection<Request> requests) {
        Collection<Address> suspectedMembers = new LinkedHashSet<>(requests.size());
        for (Request req : requests)
            if (req.type == Request.SUSPECT)
                suspectedMembers.add(req.mbr);

        if (suspectedMembers.isEmpty())
            return;

        for (Address mbr : suspectedMembers)
            if (!suspected_mbrs.contains(mbr))
                suspected_mbrs.add(mbr);

        if (wouldIBeCoordinator()) {
            log.debug("%s: members are %s, coord=%s: I'm the new coord !", gms.local_addr, gms.members, gms.local_addr);

            gms.becomeCoordinator();
            for (Address mbr : suspected_mbrs) {
                gms.getViewHandler().add(new Request(Request.SUSPECT, mbr, true));
                gms.ack_collector.suspect(mbr);
            }
            suspected_mbrs.clear();
        }
    }
```
可以看到这里只处理Request.SUSPECT类型的请求。
suspected_mbrs里面里面添加。
需要关注的是wouldIBeCoordinator()方法。
直接看。
```
boolean wouldIBeCoordinator() {
        List<Address> mbrs = gms.computeNewMembership(gms.members.getMembers(), null, null, suspected_mbrs);
        if (mbrs.size() < 1) return false;
        Address new_coord = mbrs.get(0);
        return gms.local_addr.equals(new_coord);
    }
```
这里主要是从当前节点的视图里面剔除suspected_mbrs列表里面的节点，然后获取第一个节点列表，判断是否是自己。如果是自己就说明自己成为协调者。
然后执行gms.becomeCoordinator();
升级成为协调者的方法。
这里的升级方法是。
```
public void becomeCoordinator() {
        CoordGmsImpl tmp = (CoordGmsImpl) impls.get(COORD);
        if (tmp == null) {
            tmp = new CoordGmsImpl(this);
            impls.put(COORD, tmp);
        }
        try {
            first_view_sent = false;
            tmp.init();
        } catch (Exception e) {
            log.error(Util.getMessage("ExceptionSwitchingToCoordinatorRole"), e);
        }
        setImpl(tmp);
    }
```
也就是说升级成为协调者只是替换GMS里面的impl
最后依旧是
```
gms.getViewHandler().add(new Request(Request.SUSPECT, mbr, true));
                gms.ack_collector.suspect(mbr);
```
和协调者收到Request.SUSPECT的处理一样。
所以真正的问题还是得看协调者的CoordGmsImpl.handleMembershipChange方法

从这里可以知道更多的是。当一个主节点宕机之后（长时间没法发心跳）这时候，每个一般节点由于都收到底层发上来的Request.SUSPECT事件然后都会变成协调者。
那么就有一个问题就是，最终怎么确定唯一的协调者的。
这个后面再分析。

```
public void handleMembershipChange(Collection<Request> requests) {  
        ...
        for (Request req : requests) {
            switch (req.type) {
                ....
                case Request.SUSPECT:
                    suspected_mbrs.add(req.mbr);
                    break;
            }
        }
        ....
        View new_view = gms.getNextView(new_mbrs, leaving_mbrs, suspected_mbrs);
        ....
        gms.castViewChange(new_view, null, new_mbrs);
        ....
    }
```
大概的源码就是这样，因为这里handleMembershipChange是处理JOIN，LEAVE，JOIN_WITH_STATE_TRANSFER，SUSPECT等各种视图合并的，所以比较复杂。不过对于SUSPECT来说只有两句。

```
View new_view = gms.getNextView(new_mbrs, leaving_mbrs, suspected_mbrs);
        gms.castViewChange(new_view, null, new_mbrs);
```

生成一个new_view 然后调用gms.castViewChange进行广播。
这里的新view的生成规则是。
自己本地视图里面保存的添加上新加入的剔除离开的和怀疑的对象。具体见DefaultMembershipPolicy
```
public List<Address> getNewMembership(final Collection<Address> current_members, final Collection<Address> joiners,
                                              final Collection<Address> leavers, final Collection<Address> suspects) {
            Membership mbrs = new Membership(current_members).remove(leavers).remove(suspects).add(joiners);
            return mbrs.getMembers();
        }
```
然后重点关注的应该是GMS.castViewChange方法。


## 涉及到的对象
### Request.JOIN

###  Request.LEAVE
  主线1：
    GMS.down.Event.DISCONNECT ->
    CoordGmsImpl.leave ->
    ViewHandler.add(Request.LEAVE) ->
    GMS.process(Request.LEAVE) ->
    CoordGmsImpl.handleMembershipChange(Request.LEAVE)  
    
    
主线2: 
JChannel.disconnect ->
JChannel.down(Event.DISCONNECT) ->
GMS.up(GmsHeader.LEAVE_REQ) ->
ViewHandler.add(Request.LEAVE,false) ->

### Request.SUSPECT 
   FD_ALL2.TimeoutChecker.run ->
   FD_ALL2.suspect   ->
   GMS.up(Event.SUSPECT) ->
   ViewHandler.add(Request.SUSPECT) ->
### ViewHandler


## Q & A
+ fd系列协议会对视图刷新有什么影响？
+ 节点启动的整体流程是什么？
+ 怎么确定一个节点已经是离开了？
+ 一个节点已经确认离开之后会进行什么处理？


