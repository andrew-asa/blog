# jgroup 缓存

## 启动
```
public void start() throws Exception {
        if (hash_function_factory != null) {
            hash_function = hash_function_factory.create();
        }
        if (hash_function == null)
            hash_function = new ConsistentHashFunction<>();

        ch = new JChannel(props);
        disp = new RpcDispatcher(ch, null, this, this);
        RpcDispatcher.Marshaller marshaller = new CustomMarshaller();
        disp.setRequestMarshaller(marshaller);
        disp.setResponseMarshaller(marshaller);
        disp.setMethodLookup(new MethodLookup() {
            public Method findMethod(short id) {
                return methods.get(id);
            }
        });

        ch.connect(cluster_name);
        local_addr = ch.getAddress();
        view = ch.getView();
        timer = ch.getProtocolStack().getTransport().getTimer();

        l2_cache.addChangeListener(this);
    }
```
只需关注采用RpcDispatcher作为消息发送器

## 元素添加
put(K key, V val) 
=> put(K key, V val, short repl_count, long timeout)
这里的repl_count表示当前元素存储到多少个节点上面
```
public void put(K key, V val, short repl_count, long timeout, boolean synchronous) {
        if (repl_count == 0) {
            if (log.isWarnEnabled())
                log.warn("repl_count of 0 is invalid, data will not be stored in the cluster");
            return;
        }
        mcastPut(key, val, repl_count, timeout, synchronous);
        if (l1_cache != null && timeout >= 0)
            l1_cache.put(key, val, timeout);
    }
```
异步向其它节点进行广播同时往l1_cache缓存里面放入当前元素。
=>mcastPut(K key, V val, short repl_count, long caching_time, boolean synchronous) 
```
private void mcastPut(K key, V val, short repl_count, long caching_time, boolean synchronous) {
        try {
            ResponseMode mode = synchronous ? ResponseMode.GET_ALL : ResponseMode.GET_NONE;
            disp.callRemoteMethods(null, new MethodCall(PUT, key, val, repl_count, caching_time),
                    new RequestOptions(mode, call_timeout));
        } catch (Throwable t) {
            if (log.isWarnEnabled())
                log.warn("put() failed", t);
        }
    }
```
可以看到只是异步向channel里面广播rpc方法。
节点接收到消息后的处理方法直接看RpcDispatcher的handle方法。
```
public Object handle(Message req) throws Exception {
        if (server_obj == null) {
            log.error(Util.getMessage("NoMethodHandlerIsRegisteredDiscardingRequest"));
            return null;
        }

        if (req == null || req.getLength() == 0) {
            log.error(Util.getMessage("MessageOrMessageBufferIsNull"));
            return null;
        }

        Object body = req_marshaller != null ?
                req_marshaller.objectFromBuffer(req.getRawBuffer(), req.getOffset(), req.getLength()) : req.getObject();

        if (!(body instanceof MethodCall))
            throw new IllegalArgumentException("message does not contain a MethodCall object");

        MethodCall method_call = (MethodCall) body;

        if (log.isTraceEnabled())
            log.trace("[sender=%s], method_call: %s", req.getSrc(), method_call);

        if (method_call.getMode() == MethodCall.ID) {
            if (method_lookup == null)
                throw new Exception(String.format("MethodCall uses ID=%d, but method_lookup has not been set", method_call.getId()));
            Method m = method_lookup.findMethod(method_call.getId());
            if (m == null)
                throw new Exception("no method found for " + method_call.getId());
            method_call.setMethod(m);
        }

        return method_call.invoke(server_obj);
    }
```
主要关注
method_call.invoke(server_obj)
这里的server_obj即一开始的ReplCache
直接看MethodCall.invoke方法

```
public Object invoke(Object target) throws Exception {
        if (target == null)
            throw new IllegalArgumentException("target is null");

        Class cl = target.getClass();
        Method meth = null;

        switch (mode) {
            case METHOD:
                if (this.method != null)
                    meth = this.method;
                break;
            case TYPES:
                meth = getMethod(cl, method_name, types);
                break;
            case ID:
                meth = lookup != null ? lookup.findMethod(method_id) : null;
                break;
            default:
                if (log.isErrorEnabled()) log.error("mode " + mode + " is invalid");
                break;
        }

        if (meth != null) {
            try {
                return meth.invoke(target, args);
            } catch (InvocationTargetException target_ex) {
                Throwable exception = target_ex.getTargetException();
                if (exception instanceof Error) throw (Error) exception;
                else if (exception instanceof RuntimeException) throw (RuntimeException) exception;
                else if (exception instanceof Exception) throw (Exception) exception;
                else throw new RuntimeException(exception);
            }
        } else {
            throw new NoSuchMethodException(method_name);
        }
    }
```

可以看到主要是找到Method方法并进行调用，可以看到这里找method方法支持多种模式，默认是case TYPES 模式，直接看TYPES模式的查找
先看ReplCache一开始的初始化

```
static {
        try {
            methods.put(PUT, ReplCache.class.getMethod("_put",
                    Object.class,
                    Object.class,
                    short.class,
                    long.class));
            methods.put(PUT_FORCE, ReplCache.class.getMethod("_put",
                    Object.class,
                    Object.class,
                    short.class,
                    long.class, boolean.class));
            methods.put(GET, ReplCache.class.getMethod("_get",
                    Object.class));
            methods.put(REMOVE, ReplCache.class.getMethod("_remove", Object.class));
            methods.put(REMOVE_MANY, ReplCache.class.getMethod("_removeMany", Set.class));
        } catch (NoSuchMethodException e) {
            throw new RuntimeException(e);
        }
    }
```

再看put的时候构造的事件

```
new MethodCall(PUT, key, val, repl_count, caching_time),
                    new RequestOptions(mode, call_timeout)
```
可以知道
ReplCache的put方法在本地调用put方法，在远程rpc调用_put方法。get,remove同理
直接看_put方法。
V _put(K key, V val, short repl_count, long timeout)
=> V _put(K key, V val, short repl_count, long timeout, boolean force)

```
public V _put(K key, V val, short repl_count, long timeout, boolean force) {

        if (!force) {

            // check if we need to host the data
            boolean accept = repl_count == -1;

            if (!accept) {
                if (view != null && repl_count >= view.size()) {
                    accept = true;
                } else {
                    List<Address> selected_hosts = hash_function != null ? hash_function.hash(key, repl_count) : null;
                    if (selected_hosts != null) {
                        if (log.isTraceEnabled())
                            log.trace("local=" + local_addr + ", hosts=" + selected_hosts);
                        for (Address addr : selected_hosts) {
                            if (addr.equals(local_addr)) {
                                accept = true;
                                break;
                            }
                        }
                    }
                    if (!accept)
                        return null;
                }
            }
        }

        if (log.isTraceEnabled())
            log.trace("_put(" + key + ", " + val + ", " + repl_count + ", " + timeout + ")");

        Value<V> value = new Value<>(val, repl_count);
        Value<V> retval = l2_cache.put(key, value, timeout);

        if (l1_cache != null)
            l1_cache.remove(key);

        notifyChangeListeners();

        return retval != null ? retval.getVal() : null;
    }
```
这里的force在目前的上层调用永远为false
通过hash_function函数获取保存的元素的节点列表如果节点列表包含当前节点则往l2_cache 二级缓存里面存放元素，同时删除l1_cache缓存里面的当前元素。因为l1_cache主要是上一次操作是否在当前节点。而如果是广播备份的保存的元素主要是保存到l2_cache里面。
至此put的操作解析完成。

## 查找
直接看get方法。
```
public V get(K key) {

        // 1. Try the L1 cache first
        if (l1_cache != null) {
            V val = l1_cache.get(key);
            if (val != null) {
                if (log.isTraceEnabled())
                    log.trace("returned value " + val + " for " + key + " from L1 cache");
                return val;
            }
        }

        // 2. Try the local cache
        Cache.Value<Value<V>> val = l2_cache.getEntry(key);
        Value<V> tmp;
        if (val != null) {
            tmp = val.getValue();
            if (tmp != null) {
                V real_value = tmp.getVal();
                if (real_value != null && l1_cache != null && val.getTimeout() >= 0)
                    l1_cache.put(key, real_value, val.getTimeout());
                return tmp.getVal();
            }
        }

        // 3. Execute a cluster wide GET
        try {
            RspList<Object> rsps = disp.callRemoteMethods(null,
                    new MethodCall(GET, key),
                    new RequestOptions(ResponseMode.GET_ALL, call_timeout));
            for (Rsp rsp : rsps.values()) {
                Object obj = rsp.getValue();
                if (obj == null || obj instanceof Throwable)
                    continue;
                val = (Cache.Value<Value<V>>) rsp.getValue();
                if (val != null) {
                    tmp = val.getValue();
                    if (tmp != null) {
                        V real_value = tmp.getVal();
                        if (real_value != null && l1_cache != null && val.getTimeout() >= 0)
                            l1_cache.put(key, real_value, val.getTimeout());
                        return real_value;
                    }
                }
            }
            return null;
        } catch (Throwable t) {
            if (log.isWarnEnabled())
                log.warn("get() failed", t);
            return null;
        }
    }
```
先从当前节点的一二级缓存里面查找，如果查找到，且没有过期则直接进行返回。如果没有查找到则进行广播进行远程调用。
远程调用同上面分析直接看_get方法
```
 public Cache.Value<Value<V>> _get(K key) {
        if (log.isTraceEnabled())
            log.trace("_get(" + key + ")");
        return l2_cache.getEntry(key);
    }
```     
直接从二级缓存里面查找，这里之所有不从l1_cache里面查找，因为是想确保查找最新的。

## 删除   

```
public void remove(K key, boolean synchronous) {
        try {
            disp.callRemoteMethods(null, new MethodCall(REMOVE, key),
                    new RequestOptions(synchronous ? ResponseMode.GET_ALL : ResponseMode.GET_NONE, call_timeout));
            if (l1_cache != null)
                l1_cache.remove(key);
        } catch (Throwable t) {
            if (log.isWarnEnabled())
                log.warn("remove() failed", t);
        }
    }
```    
rpc广播调用远程方法，进行删除本地缓存。
远程rpc remove方法
```
public V _remove(K key) {
        if (log.isTraceEnabled())
            log.trace("_remove(" + key + ")");
        Value<V> retval = l2_cache.remove(key);
        if (l1_cache != null)
            l1_cache.remove(key);
        notifyChangeListeners();
        return retval != null ? retval.getVal() : null;
    }
```    
直接删除缓存。
            