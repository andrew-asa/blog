# VERIFY_SUSPCET
VERIFY_SUSPCET协议通过ping的方式再次确认可疑节点是否已终止。如已终止，则向上发送SUSPECT通知，将该节点移除；否则，直接丢弃SUSPECT消息。
默认情况下，向SUSPECT节点发送一个确认心跳消息，等待2秒钟。如果SUSPECT节点响应则丢弃SUSPECT消息，如无响应则认定节点丢失，向上发送SUSPECT通知。




## 参数
字段 | 默认值 | 含义
--- | --- | ---
bind_addr |  null |ICMP ping的接口。如果use_icmp为true，则使用以下特殊值：GLOBAL，SITE_LOCAL，LINK_LOCAL和NON_LOOPBACKK
num_msgs |  1 | 发送给可疑成员的验证心跳数
timeout |  2000 | 等待来自可疑成员的响应的毫秒数
use_icmp | false  | 使用InetAddress.isReachable（）来验证可疑成员而不是常规消息
use_mcast_rsps |  false | 将I_AM_NOT_DEAD消息作为多播而不是多个单播发送回来（默认为false）(在收到VerifyHeader.ARE_YOU_DEAD消息的时候是否进行广播)

## 源码
在FD_ALL2协议解析中，可以知道，一定时间收不到某个节点的心跳信息的时候会在suspect方法中会调用
```
up_prot.up(new Event(Event.SUSPECT, suspect));
down_prot.down(new Event(Event.SUSPECT, suspect));
```
目前core的协议栈大概是这样配置的。
```
    <TCP/>
    <TCPPING />
    <MERGE3 />
    <FD_SOCK/>
    <FD_ALL2 interval="3000" timeout="15000"/>
    <VERIFY_SUSPECT timeout="1500"/>
    <BARRIER/>
    <pbcast.NAKACK2 />
    ...      
```
在FD_ALL2里面
up就是VERIFY_SUSPECT协议
down就是FD_SOCK协议
VERIFY_SUSPECT.up(VERIFY_SUSPEC)这就是事件的起始位置。

所以直接看VERIFY_SUSPECT收到up方法收到Event.SUSPECT事件是怎么处理的。

```
public Object up(Event evt) {
        switch (evt.getType()) {

            case Event.SUSPECT:  // it all starts here ...
                Address suspected_mbr = (Address) evt.getArg();
                if (suspected_mbr == null) {
                    if (log.isErrorEnabled()) log.error(Util.getMessage("SuspectedMemberIsNull"));
                    return null;
                }

                if (local_addr != null && local_addr.equals(suspected_mbr)) {
                    if (log.isTraceEnabled())
                        log.trace("I was suspected; ignoring SUSPECT message");
                    return null;
                }

                if (!use_icmp)
                    verifySuspect(suspected_mbr);
                else
                    verifySuspectWithICMP(suspected_mbr);
                return null;  // don't pass up; we will decide later (after verification) whether to pass it up


            case Event.MSG:
                Message msg = (Message) evt.getArg();
                VerifyHeader hdr = (VerifyHeader) msg.getHeader(this.id);
                if (hdr == null)
                    break;
                switch (hdr.type) {
                    case VerifyHeader.ARE_YOU_DEAD:
                        if (hdr.from == null) {
                            if (log.isErrorEnabled()) log.error(Util.getMessage("AREYOUDEADHdrFromIsNull"));
                        } else {
                            Message rsp;
                            Address target = use_mcast_rsps ? null : hdr.from;
                            for (int i = 0; i < num_msgs; i++) {
                                rsp = new Message(target).setFlag(Message.Flag.INTERNAL)
                                        .putHeader(this.id, new VerifyHeader(VerifyHeader.I_AM_NOT_DEAD, local_addr));
                                down_prot.down(new Event(Event.MSG, rsp));
                            }
                        }
                        return null;
                    case VerifyHeader.I_AM_NOT_DEAD:
                        if (hdr.from == null) {
                            if (log.isErrorEnabled()) log.error(Util.getMessage("IAMNOTDEADHdrFromIsNull"));
                            return null;
                        }
                        unsuspect(hdr.from);
                        return null;
                }
                return null;


            case Event.CONFIG:
                if (bind_addr == null) {
                    Map<String, Object> config = (Map<String, Object>) evt.getArg();
                    bind_addr = (InetAddress) config.get("bind_addr");
                }
        }
        return up_prot.up(evt);
    }
```

可以看到
1：收到Event.SUSPECT 事件
如果配置了use_icmp就用verifySuspectWithICMP方法进行验证节点是否存活。
这里的verifySuspectWithICMP本质就是使用InetAddress..isReachable方法进行判断。
因为目前的core里面没有配置这个参数，所以这种方式的优缺点先忽略

直接看verifySuspect方法
```
void verifySuspect(Address mbr) {
        Message msg;
        if (mbr == null) return;

        addSuspect(mbr);

        startTimer(); // start timer before we send out are you dead messages

        // moved out of synchronized statement (bela): http://jira.jboss.com/jira/browse/JGRP-302
        if (log.isTraceEnabled()) log.trace("verifying that " + mbr + " is dead");

        for (int i = 0; i < num_msgs; i++) {
            msg = new Message(mbr).setFlag(Message.Flag.INTERNAL)
                    .putHeader(this.id, new VerifyHeader(VerifyHeader.ARE_YOU_DEAD, local_addr));
            down_prot.down(new Event(Event.MSG, msg));
        }
    }
```
调用addSuspect和startTimer方法，然后调用
down协议发送VerifyHeader.ARE_YOU_DEAD询问信息。
这里的down协议就是就是MERGE3最终会调用TCP发送到指定的地址。

接着看addSuspect方法。
```
protected boolean addSuspect(Address suspect) {
        if (suspect == null)
            return false;
        synchronized (suspects) {
            for (Entry entry : suspects) // check for duplicates
                if (entry.suspect.equals(suspect))
                    return false;
            suspects.add(new Entry(suspect, System.currentTimeMillis() + timeout));
            return true;
        }
    }
```
就是往suspects里面放一个Entry对象。
先看suspects对象是什么

```
protected final DelayQueue<Entry> suspects = new DelayQueue<>();
```
一个延迟队列
再看Entry对象
```
protected class Entry implements Delayed {
        protected final Address suspect;
        protected final long target_time;

        public Entry(Address suspect, long target_time) {
            this.suspect = suspect;
            this.target_time = target_time;
        }

        public int compareTo(Delayed o) {
            Entry other = (Entry) o;
            long my_delay = getDelay(TimeUnit.MILLISECONDS), other_delay = other.getDelay(TimeUnit.MILLISECONDS);
            return my_delay < other_delay ? -1 : my_delay > other_delay ? 1 : 0;
        }

        public long getDelay(TimeUnit unit) {
            long delay = target_time - System.currentTimeMillis();
            return unit.convert(delay, TimeUnit.MILLISECONDS);
        }

        public String toString() {
            return suspect + ": " + target_time;
        }
    }
```
一个实现Delayed接口的延时对象
接着看startTimer方法
```
protected synchronized void startTimer() {
        if (timer == null || !timer.isAlive()) {
            timer = getThreadFactory().newThread(this, "VERIFY_SUSPECT.TimerThread");
            timer.setDaemon(true);
            timer.start();
        }
    }
```
所以直接看
VERIFY_SUSPECT.run方法。
```
public void run() {
        while (!suspects.isEmpty() && timer != null) {
            try {
                Entry entry = suspects.poll(timeout * 2, TimeUnit.MILLISECONDS);
                if (entry != null) {
                    if (log.isTraceEnabled())
                        log.trace(entry.suspect + " is dead (passing up SUSPECT event)");
                    up_prot.up(new Event(Event.SUSPECT, entry.suspect));
                }
            } catch (InterruptedException e) {
            }
        }
    }
```
如果能从suspects获取一个对象则直接
向up协议发送Event.SUSPECT事件。
这里的up协议就是
BARRIER
NAKACK2 等协议，也就是最终确认了改节点是值得怀疑的。
要理解为什么拿出来就是确定怀疑的是还要了解DelayQueue这个对象。
DelayQueue的poll方法会返回一个已经过期的时间，而这个过期的算法允许Delayed自定义
这里就是上面的long getDelay(TimeUnit unit)
方法。
即超过从加入到suspects队列里面timeout时间内就可以从这里面队列里面并表示该怀疑节点已经被确认值得怀疑。

所以这里重要的是看节点什么时候可以从suspects队列里面剔除。

接着看up方法里面收到
VerifyHeader.ARE_YOU_DEAD消息（即上面说的，当收到FD_ALL2的Event.SUSPECT时间之后发送的消息。）
的处理。
```
rsp = new Message(target).setFlag(Message.Flag.INTERNAL)
                                        .putHeader(this.id, new VerifyHeader(VerifyHeader.I_AM_NOT_DEAD, local_addr));
                                down_prot.down(new Event(Event.MSG, rsp));
```
直接会送一条VerifyHeader.I_AM_NOT_DEAD消息。
当发送方接受到VerifyHeader.I_AM_NOT_DEAD消息之后
```
case VerifyHeader.I_AM_NOT_DEAD:
                        if (hdr.from == null) {
                            if (log.isErrorEnabled()) log.error(Util.getMessage("IAMNOTDEADHdrFromIsNull"));
                            return null;
                        }
                        unsuspect(hdr.from);
```
直接看unsuspect方法
```
public void unsuspect(Address mbr) {
        boolean removed = mbr != null && removeSuspect(mbr);
        if (removed) {
            if (log.isTraceEnabled()) log.trace("member " + mbr + " was unsuspected");
            down_prot.down(new Event(Event.UNSUSPECT, mbr));
            up_prot.up(new Event(Event.UNSUSPECT, mbr));
        }
    }
```
removeSuspect方法
```
protected boolean removeSuspect(Address suspect) {
        if (suspect == null)
            return false;
        boolean retval = false;
        synchronized (suspects) {
            for (Iterator<Entry> it = suspects.iterator(); it.hasNext(); ) {
                Entry entry = it.next();
                if (entry.suspect.equals(suspect)) {
                    it.remove();
                    retval = true; // don't break, possibly remove more (2nd line of defense)
                }
            }
        }
        return retval;
    }
```
这里就是直接从队列里面删除怀疑节点了。

总结起来就是
1：VERIFY_SUSPECT收到Event.SUSPECT消息则往怀疑队列里面放一个节点信息，同时向节点发送VerifyHeader.ARE_YOU_DEAD消息
2：被怀疑消息收到VerifyHeader.ARE_YOU_DEAD消息之后回送
VerifyHeader.I_AM_NOT_DEAD消息
3：如果超过timeout时间没有接收到VerifyHeader.I_AM_NOT_DEAD则向协议栈上层继续传送Event.SUSPECT消息。
4：如果超时时间内收到某个节点的VerifyHeader.I_AM_NOT_DEAD消息则向上，向下协议栈发送Event.UNSUSPECT消息。

## Q
1：同样没能回答视图什么时候进行改变的问题。



