# jgroup 锁相关
## 锁的类型

## 涉及到的对象
+ LockService
    + 工厂服务类，创建锁的入口
+ LockImpl
+ LockInfo
+ Locking
+ ServerLock

## 客户端
+ 请求锁定
    + ClientLock.lock ==> acquire ==> sendGrantLockRequest 向协调者发送
Event.MSG类信息里面的mes的信息体位Locking.Request
Request的type类型为GRANT_LOCK同时调用ClientLock.acquire方法
本质上是调用ClientLock的wait方法等待别的线程通知，具体见

```
protected synchronized void acquire(boolean throwInterrupt) throws InterruptedException {
            if(acquired)
                return;
            if(throwInterrupt && Thread.interrupted())
                throw new InterruptedException();
            owner=getOwner();
            sendGrantLockRequest(name, lock_id, owner, 0, false);
            boolean interrupted=false;
            while(!acquired) {
                try {
                    this.wait();
                }
                catch(InterruptedException e) {
                    if(throwInterrupt && !acquired) {
                        _unlock(true);
                        throw e;
                    }
                    // If we don't throw exceptions then we just set the interrupt flag and let it loop around
                    interrupted=true;
                }
            }
            if(interrupted)
                Thread.currentThread().interrupt();
        }
``` 

    +   如果成功获取锁则会收到Request的type类型为LOCK_GRANTED的信息最终会调用ClientLock.lockGranted方法具体见这里的lock_id 是线程id 


```
 protected synchronized void lockGranted(int lock_id) {
            if(this.lock_id != lock_id) {
                log.error("discarded LOCK-GRANTED response with lock-id=" + lock_id + ", my lock-id=" + this.lock_id);
                return;
            }
            acquired=true;
            this.notifyAll();
        }
```

    + 如果获取失败则会收到Request的type类型为LOCK_DENIED的信息最终会调用
ClientLock.lockDenied方法具体见这里的lock_id 是线程id 
具体参见

```
protected synchronized void lockDenied(int lock_id) {
            if(this.lock_id != lock_id) {
                log.error("discarded LOCK-DENIED response with lock-id=" + lock_id + ", my lock_id=" + this.lock_id);
                return;
            }
            denied=true;
            this.notifyAll();
        }
```
从上面的代码可以总结即，对于请求锁的客户端来说
1：向协调者发送锁授权请求。
2: 休眠等待
3：接收到协调者发来的被授权或者被拒绝请求将被唤醒。如果是被授予则进行下面的流程
如果是被拒绝则继续休眠。

## 服务器端
+ 请求锁定
    + 协调者接收到客户端的Locking.Request.GRANT_LOCK 请求会查找或创建（根据锁的名字） ServerLock对象调用ServerLock.handleRequest方法
    + 如果锁（注意这里的锁是的标识是由 锁的名字+锁的线程id构成）没有被获取获取或者说获取者跟请求者是同一个则发送一个Type.LOCK_GRANTED锁被授予的响应。
具体代码段见

```
 protected Response handleRequest(Request req) {
            switch(req.type) {
                case GRANT_LOCK:
                    if(current_owner == null) {
                        setOwner(req.owner);
                        return new Response(Type.LOCK_GRANTED, req.owner, req.lock_name, req.lock_id);
                    }
                    if(current_owner.equals(req.owner))
                        return new Response(Type.LOCK_GRANTED, req.owner, req.lock_name, req.lock_id);

                    if(req.is_trylock && req.timeout <= 0)
                        return new Response(Type.LOCK_DENIED, req.owner, req.lock_name, req.lock_id);
                    addToQueue(req);
                    break;
               ....
            }

            return processQueue();
        }
}
```
从上面可以看成只有明确获取了锁才会立马返回一个Type.LOCK_GRANTED的消息，或者是尝试获取锁且不设置超时等待但是锁已经被别的请求获取了才会立马返回一个LOCK_DENIED消息
别的分支都是往队列里面添加消息然后再末尾的时候进行处理消息。
processQueue 主要的作用是再次尝试获取锁具体见如下。
```
 protected Response processQueue() {
            // 说明被人快一步了，能走到这边只可能是锁已经被别的owner拥有
            if(current_owner != null)
                return null;
            // 当前锁的拥有者为null才分配
            while(!queue.isEmpty()) {
                Request req=queue.remove(0);
                if(req.type == Type.GRANT_LOCK) {
                    setOwner(req.owner);
                    return new Response(Type.LOCK_GRANTED, req.owner, req.lock_name, req.lock_id);
                }
            }
            return null;
        }
```
从上面可以看出给客户端的相应有可能是明确的给出已经授权，禁止授权，或者为null
看一下协调者怎么处理。
```
protected void handleLockRequest(Request req) {
        Response rsp=null;
        Lock lock=_getLock(req.lock_name);
        lock.lock();
        try {
            ...
            rsp=server_lock.handleRequest(req);
            ...
        }
        finally {
            lock.unlock();
        }
        if(rsp != null)
            sendLockResponse(rsp.type, rsp.owner, rsp.lock_name, rsp.lock_id);
    }
```
上面的erver_lock就是协调者端对一个lock处理的对象，可以看到就是如果有回执消息就直接发送。如果没有则什么都不做。
对于客户端来说，null 和Type.LOCK_DENIED 本质是一样的，即会一直等待。对于LOCK_GRANTED 则会尽心下面的操作。

## 疑问 & 可靠性
+ 如果客户端发送到了锁请求，协调者也收到了锁请求且授予了客户端但是被授权的消息没有被客户端接收到，客户端是否会一直等待?这个问题可以转换成另一个问题即节点间的通讯是否是可靠的，或者说消息交付是否是可靠，如果不可靠怎么保证这个锁机制是正确的？
+ 上面的问题是否就是jgroup的udp/tcp通信问题？
+ 上面的两个问题留在jgroup-tp 通讯协议里面讨论。

## Request
request type 类型| 发送方作用 | 接收方作用
--- | ---| --- | ---
GRANT_LOCK| 要求获得锁定|  |
GRANT_LOCK| 成功锁定获取时响应GRANT_LOCK的发送者|  || 要求获得锁定|  |
LOCK_DENIED|在失败的锁定获取时响应GRANT_LOCK的发送者|


## 协议
## CENTRAL_LOCK 中央锁，即锁的获取者有协调者给出。
+ 让集群的当前协调器授予锁定，因此每个节点都必须与协调器通信以获取或释放锁定。不同节点对同一锁的锁定请求按接收顺序处理
+ 协调员维护一个锁表。为了防止丢失谁拥有哪个锁的知识，协调器可以将锁信息推送到num_backups定义的多个备份。如果num_backups为0，则不会发生锁定信息的复制。
+ 如果num_backups大于0，则协调器将有关已获取和已释放锁的信息推送到所有备份节点。
+ 拓扑更改可能会创建新的备份节点，并且锁定信息将推送到成为新备份节点的位置。
+ CENTRAL_LOCK的优点是所有锁定请求都在群集中以相同的顺序授予，而PEER_LOCK则不是这种情况。
+ 参数
    + num_backups协调器的备份数量。, 服务器锁也会复制到这些节点
    + use_thread_id_for_lock_owner 默认情况下，锁所有者是address + thread-id。如果为false，我们只使用节点的地址
+ 获取到锁的时候将会锁定它，接着释放它。如果成员N已经得到了锁L，新的节点NN获取锁X会被阻塞，直到N释放掉该锁，或者N离开该集群或崩溃。

## PEER_LOCK 点对点锁，即锁的获取拥有所有人推荐出来。
+ 通过联系所有群集节点获取锁定，并且只有在所有非故障群集节点（对等方）授予锁定时，锁定获取才会成功。
+ 除非使用总订单配置（例如，基于org.jgroups.protocols.SEQUENCER），否则可以以不同的顺序接收来自不同发送方的对相同资源的锁请求，因此可能发生死锁。例：
+ Nodes A 和 B
+ A 和 B 请求锁 lock(X) 同一时间
+ A 接受到请求 L(X,A) 然后是 L(X,B): locks X(A), 然后查询 L(X,B)
+ B 接受到 L(X,B) 然后是 L(X,A): locks X(B), 然后查询 L(X,A)
+ 要获得锁定，我们需要来自A和B的锁定授权，但这绝不会发生。要解决此问题，请将SEQUENCER添加到配置中，以便在A和B处以相同的全局顺序接收所有锁定请求，或者如果无法获取锁，则使用java.util.concurrent.locks.Lock.tryLock（long，javaTimeUnit）重试。

## 参考连接
+ [http://jgroups.org/manual/index.html#PEER_LOCK](http://jgroups.org/manual/index.html#PEER_LOCK)
+ [https://blog.csdn.net/tengdazhang770960436/article/details/49472169](https://blog.csdn.net/tengdazhang770960436/article/details/49472169)
+ [https://www.jianshu.com/p/155260c8af6c](https://www.jianshu.com/p/155260c8af6c)
+ [深入分析Synchronized原理 https://juejin.im/post/5b4eec7df265da0fa00a118f](https://juejin.im/post/5b4eec7df265da0fa00a118f)
