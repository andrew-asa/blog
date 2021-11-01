# 基于redis的锁

## 起因
1：jgroup 的锁目前除了很多问题，脆弱，比如在视图合并事件中，目前的做法是释放当前客户端的所有锁。
2：目前的系统采用的jgroup的中央锁，采用的也是由一个节点来分配锁。
3：redis号称是单线程执行脚本（真正的代码并没有看过，而且目前使用上并没有出现过问题，所以暂时这样认为）

## redis 里面存的数据结构
1: 用一个hash来存放一个锁
2: hash的key 为 redis存储前缀+ 锁的名称 => (下面表示锁名称)
3: lock的标识为节点id + 线程id =>（下面表示锁id）
4: 
## 具体实现
+ lock 锁住
直接看关键redis lua脚本

```
if (redis.call('exists', KEYS[1]) == 0) then
                    redis.call('hset', KEYS[1], ARGV[2], 1);
                    redis.call('pexpire', KEYS[1], ARGV[1]);
                    return nil;
                end;
                if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then
                    redis.call('hincrby', KEYS[1], ARGV[2], 1);
                    redis.call('pexpire', KEYS[1], ARGV[1]);
                    return nil;
                end;
                return redis.call('pttl', KEYS[1]);
``` 
KEY[1] => 锁的名称
ARGV[1] => key的过期时间
ARGV[2] = > 锁的id

1: 如果锁的名称的key值不存在，表示没有被锁，则进行设置相应的值，同时设置过期时间并返回nil表示成功获取锁
2：如果if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) 表示
是当前节点，当前线程拥有锁，则拥有数自增同时返回nil表示成功获取锁
3：最后的情况就表示当前锁是被别的节点，别的线程拥有，则直接返回过期时间。

从redis里面获取值之后的代码处理直接看代码

```
public void lockInterruptibly(long leaseTime, TimeUnit unit) throws InterruptedException {

        long threadId = Thread.currentThread().getId();
        Long ttl = tryAcquire(leaseTime, unit, threadId);
        if (ttl == null) {
            return;
        }
        // 订阅锁频道
        subscribe(threadId);
        try {
            while (true) {
                ttl = tryAcquire(leaseTime, unit, threadId);
                // 锁被获取
                if (ttl == null) {
                    break;
                }
                // 等待锁被释放通知
                if (ttl >= 0) {
                    getEntry(threadId).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
                } else {
                    getEntry(threadId).getLatch().acquire();
                }
            }
        } finally {
            unsubscribe(threadId);
        }
    }
```

从上面的lua脚本可以知道，从redis中返回的只有两种情况
1：null -> 表示成功获取锁
2：ttl -> 表示这个锁什么时候过期

所以就对应两种处理
1： null 的时候则设置定时器，设置不断的往redis里面更新key的过期时间。
具体的代码在tryAcquired的scheduleExpirationRenewal方法里面
```
private Long tryAcquire(long leaseTime, TimeUnit unit, long threadId) {

        Long ret;
        if (leaseTime != -1) {
            ret = doTryAcquire(leaseTime,
                               unit,
                               threadId);
        } else {
            ret = doTryAcquire(getConfig().getLockWatchdogTimeout(),
                               TimeUnit.MILLISECONDS,
                               threadId);
        }
        if (ret == null) {
            // 成功获取锁了，该放看门狗了
            FineLoggerFactory.getLogger().debug("try acquire success name {}, thread id {}", getName(), threadId);
            scheduleExpirationRenewal(threadId);
        }
        return ret;
    }
```
定时执行的lua代码就是
```
if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then
                    redis.call('pexpire', KEYS[1], ARGV[1]);
                    return 1;
                end;
                return 0;
```
KEYS[1] -> 锁名称
ARGV[1] -> 过期时间
ARGV[2] -> 锁的id

整个成功获取锁的善后逻辑就是
1：设置一个定时器不断更新锁的过期时间，这样做的一个目的是当一个节点拥有锁之后的突然宕机能正确的释放锁。

如果没能获取锁即lock的lua脚本返回的是正整数的情况。则
1：订阅以锁名字+锁订阅channel前缀表示的锁频道。
2：并不断的进行等待。

具体的代码是
```
while (true) {
                ttl = tryAcquire(leaseTime, unit, threadId);
                // 锁被获取
                if (ttl == null) {
                    break;
                }
                // 等待锁被释放通知
                if (ttl >= 0) {
                    getEntry(threadId).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
                } else {
                    getEntry(threadId).getLatch().acquire();
                }
            }
```

这里的entry是一个信号量
等订阅频道里面有锁解锁的消息的时候回进行通知。然后再尝试获取锁。

+ unlock 解锁

直接看lua脚本
```
if (redis.call('exists', KEYS[1]) == 0) then
                    redis.call('publish', KEYS[2], ARGV[1]);
                    return 1;
                end;
                if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then
                    return nil;
                end;
                local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1);
                if (counter > 0) then
                    redis.call('pexpire', KEYS[1], ARGV[2]);
                    return 0;
                else
                    redis.call('del', KEYS[1]);
                    redis.call('publish', KEYS[2], ARGV[1]);
                    return 1;
                end;
                return nil;
```

KEYS[1] => 锁名称
KEYS[2] => 当前锁的锁频道
ARGV[1] => 发送的给频道的锁消息（这里表示解锁）
ARGV[2] => 过期时间
ARGV[3] => 线程id

总体逻辑就是，如果释放的是自己id+线程拥有的锁
则进行发布解锁消息，这时候，lock的时候
监听的锁频道就会收到解锁消息同时唤起线程。

总体的返回逻辑就是
nil -> null -> 尝试释放不是自己拥有的锁
1 -> 成功释放(没有线程拥有锁|是当前线程只lock一次)
0 -> 当前线程还有锁的次数没有被释放

redis 返回后的逻辑
```
long threadId = Thread.currentThread().getId();
        Boolean ret = doUnlock(threadId);
        if (ret == null) {
            throw new IllegalMonitorStateException("attempt to unlock lock, not locked by current thread by node id: " + nodeId + " thread-id: " + threadId);
        }
        // 成功释放锁
        if (ret) {
            cancelExpirationRenewal(null);
        }
```
nil -> null -> 尝试释放不是自己拥有的锁 -> 抛出异常
1 -> 成功释放 -> cancelExpirationRenewal取消定时更新key过期的线程。



