# 提供一个集群版ehcache
主要方案是ehcache + jgroup
jgroup 主要用于节点之间信息交互

## 启动
JGroupsCacheManagerPeerProvider的
init的方法
```
public void init() {
                channel = JChannelFactory.build(stackType);
        final String clusterName = this.getClusterName();
        this.cachePeer = new JGroupsCachePeer(this.channel, clusterName);
        //必须先初始化bootstrapManager
        initBootstrapManager();
        iniCacheManagerEventHandler();

        if (cacheManagerEventHandler != null) {
            this.cacheReceiver = new JGroupsCacheReceiver(this.cacheManager, this.bootstrapManager, this.cacheManagerEventHandler);
        } else {
            this.cacheReceiver = new JGroupsCacheReceiver(this.cacheManager, this.bootstrapManager);
        }

        this.channel.setReceiver(this.cacheReceiver);
        this.channel.setDiscardOwnMessages(true);

        try {
            this.channel.connect(clusterName);
        } catch (Exception e) {
            FineLoggerFactory.getLogger().error("Failed to connect to JGroups cluster '" + clusterName +
                    "', replication will not function. JGroups properties:\n" + this.groupProperties, e);
            this.dispose();
            return;
        }

        this.cachePeersListCache = Collections.singletonList((CachePeer) this.cachePeer);
    }
```
上面省略掉不必要的内容，只需要知道是
1: 启动一个jgroup channel
2: 采用JGroupsCacheReceiver作为节点接收信息的处理类

## 节点启动时信息同步
ehcache 主要操作的对象是Ehcache，而Ehcache 由CacheManager管理，产生，所以基于jgroup的节点启动的信息同步其实是同步CacheManager里面的信息。core里面用FineCacheManager封装了一层CacheManager。
可以自己通过定制一份FactoryConfiguration来进行定制CacheManager,其中比较重要的一个属性是BootstrapManagerProvider
如图
DefaultCacheManagerConfig
```
@Override
    public FactoryConfiguration getCacheManagerPeerProviderFactory() {

        if (ClusterBridge.isClusterMode()) {
            FactoryConfiguration fc = new FactoryConfiguration();
            fc.setClass(JGroupsCacheManagerPeerProviderFactory.class.getName());
            fc.setProperties(
                    "ProtocolStackType=" + ProtocolStackType.GENERAL_CACHE.name() +
                            "::BootstrapManagerProvider=" + DefaultBootstrapManager.class.getName());
            fc.setPropertySeparator("::");
            return fc;
        }
        return null;
    }
```

每当调用
CacheManager.addCache(Ehcache cache)生成一个ehcache的时候会调用JGroupsBootstrapCacheLoader的load(Ehcache cache)方法
```
public void load(Ehcache cache) throws RemoteCacheException {
        final JGroupsCacheManagerPeerProvider cachePeerProvider = JGroupsCacheManagerPeerProvider.getCachePeerProvider(cache);

        final BootstrapRequest bootstrapRequest = new BootstrapRequest(cache, this.asynchronous, this.maximumChunkSizeBytes);
        final BootstrapManagerProvider bootstrapManager = cachePeerProvider.getBootstrapManager();
        bootstrapManager.handleBootstrapRequest(bootstrapRequest);
    }
```
封装一个BootstrapRequest
最终会调用上面指定的BootstrapManagerProvider.handleBootstrapRequest(BootstrapRequest bootstrapRequest)方法。
接下来关注该方法。

```
public void handleBootstrapRequest(BootstrapRequest bootstrapRequest) {
        ...
        final Ehcache cache = bootstrapRequest.getCache();
        final String cacheName = cache.getName();

        final BootstrapRequest oldRequest = this.bootstrapRequests.put(cacheName, bootstrapRequest);

        ...

        final BootstrapRequestRunnable bootstrapRequestRunnable = new BootstrapRequestRunnable(bootstrapRequest);
        final Future<?> future = this.bootstrapThreadPool.submit(bootstrapRequestRunnable);

        if (!bootstrapRequest.isAsynchronous()) {
            LOG.debug("Waiting up to {}ms for BootstrapRequest of {} to complete", BOOTSTRAP_RESPONSE_MAX_TIMEOUT, cacheName);
            try {
                future.get(BOOTSTRAP_RESPONSE_MAX_TIMEOUT, TimeUnit.MILLISECONDS);
            } catch (InterruptedException e) {
                LOG.warn("Interrupted while waiting for bootstrap of " + cacheName + " to complete", e);
            } catch (ExecutionException e) {
                LOG.warn("Exception thrown while bootstrapping " + cacheName, e);
            } catch (TimeoutException e) {
                LOG.warn("Timed out waiting " + BOOTSTRAP_RESPONSE_MAX_TIMEOUT + "ms for bootstrap of " + cacheName + " to complete", e);
            }
        }
    }
```

封装一个BootstrapRequestRunnable提交到bootstrapThreadPool线程池里面。接下来看
BootstrapRequestRunnable的run方法。最终会调用其runInternal方法
```
public void runInternal() {
            final Ehcache cache = bootstrapRequest.getCache();
            final String cacheName = cache.getName();

            try {
                final List<Address> addresses = cachePeer.getOtherGroupMembers();

                if (addresses == null || addresses.size() == 0) {
                    LOG.info("There are no other nodes in the cluster to bootstrap {} from", cacheName);
                    return;
                }

                final Address localAddress = cachePeer.getLocalAddress();
                LOG.debug("Loading cache {} with local address {} from peers: {}", new Object[]{cacheName, localAddress, addresses});

                int replicationCount = 0;
                do {
                    bootstrapRequest.reset();

                    final int randomPeerNumber = BOOTSTRAP_PEER_CHOOSER.nextInt(addresses.size());
                    final Address address = addresses.remove(randomPeerNumber);

                    JGroupEventMessage event = new JGroupEventMessage(JGroupEventMessage.BOOTSTRAP_REQUEST, localAddress, null, cacheName);
                    if (LOG.isDebugEnabled()) {
                        LOG.debug("Requesting bootstrap of {} from {}", cacheName, address);
                    }

                    cachePeer.send(address, Arrays.asList(event));

                    waitForBootstrap(cacheName, address);

                    replicationCount += bootstrapRequest.getReplicationCount();

                    //Loop until the bootstrap process is complete or we've contacted all peers
                } while (bootstrapRequest.getBootstrapStatus() != BootstrapRequest.BootstrapStatus.COMPLETE && addresses.size() > 0);

                if (BootstrapRequest.BootstrapStatus.COMPLETE == bootstrapRequest.getBootstrapStatus()) {
                    LOG.info("Bootstrap for cache {} is complete, loaded {} elements", cacheName, replicationCount);
                } else {
                    LOG.info("Bootstrap for cache {} ended with status {}, loaded {} elements",
                            new Object[]{cacheName, bootstrapRequest.getBootstrapStatus(), replicationCount,});
                }
            } finally {
                final BootstrapRequest removedRequest = bootstrapRequests.remove(cacheName);
                if (removedRequest == null) {
                    LOG.warn("No BootstrapRequest for {} to remove", cacheName);
                    return;
                }

                LOG.debug("Removed {}", removedRequest);
            }
        }
```
可以看到只是简单的获取已经存在的节点列表，这里的节点列表就是启动的channel的节点列表。
获取到节点列表之后轮询发送
new JGroupEventMessage(JGroupEventMessage.BOOTSTRAP_REQUEST, localAddress, null, cacheName); 

接下来看JGroupsCacheReceiver 接收到该消息之后怎么进行处理

```
private void handleJGroupNotification(final JGroupEventMessage message) {
        final String cacheName = message.getCacheName();

        switch (message.getEvent()) {
            case JGroupEventMessage.BOOTSTRAP_REQUEST: {
              this.bootstrapManager.sendBootstrapResponse(message);
                break;
            }
            
}
```
直接调用bootstrapManager的sendBootstrapResponse方法。
这里的bootstrapManager同样是一开始指定的BootstrapManagerProvider即DefaultBootstrapManager类
```
public void sendBootstrapResponse(JGroupEventMessage message) {
        if (!this.alive) {
            LOG.warn("dispose has been called, no new BootstrapResponses will be handled");
            return;
        }

        final BootstrapResponseRunnable bootstrapResponseRunnable = new BootstrapResponseRunnable(message);
        this.bootstrapThreadPool.submit(bootstrapResponseRunnable);
    }
```
同样是生成一个BootstrapResponseRunnable线程类提交给bootstrapThreadPool
同BootstrapRequestRunnable一样直接看runInternal方法
```
public void runInternal() {
            final Address requestAddress = (Address) this.message.getSerializableKey();
            final String cacheName = this.message.getCacheName();
            final Ehcache cache = cacheManager.getEhcache(cacheName);
            
            final List<?> keys = cache.getKeys();
            if (keys == null || keys.size() == 0) {
            } else {
                final List<JGroupEventMessage> messageList = new ArrayList<JGroupEventMessage>(Math.min(keys.size(), BOOTSTRAP_CHUNK_SIZE));
                for (final Object key : keys) {
                    final Element element = cache.getQuiet(key);
                    if (element == null || element.isExpired()) {
                        continue;
                    }

                    final JGroupEventMessage groupEventMessage = new JGroupEventMessage(JGroupEventMessage.BOOTSTRAP_RESPONSE,
                            (Serializable) key, element, cacheName);

                    messageList.add(groupEventMessage);

                    if (messageList.size() == BOOTSTRAP_CHUNK_SIZE) {
                        this.sendResponseChunk(cache, requestAddress, messageList);
                        messageList.clear();
                    }

                }
                if (messageList.size() > 0) {
                    this.sendResponseChunk(cache, requestAddress, messageList);
                }
            }
            final JGroupEventMessage bootstrapCompleteMessage = new JGroupEventMessage(JGroupEventMessage.BOOTSTRAP_COMPLETE,
                    null, null, cacheName);
            cachePeer.send(requestAddress, Arrays.asList(bootstrapCompleteMessage));
        }
```
可以看到只是获取本地ehcache然后发给启动点。发送完之后发送一个JGroupEventMessage.BOOTSTRAP_COMPLETE消息

## 更新
Ehcache.put
=>Cache.putInternal
=>notifyPutInternalListeners
=>JGroupsCacheReplicator.notifyElementPut
监听器 JGroupsCacheReplicator
notifyElementPut
```
public void notifyElementPut(Ehcache cache, Element element) throws CacheException {
        if (notAlive() || !replicatePuts) {
            return;
        }

        replicatePutNotification(cache, element);
    }
```
直接看replicatePutNotification
```
private void replicatePutNotification(Ehcache cache, Element element) {
        if (!element.isKeySerializable()) {
            FineLoggerFactory.getLogger().error("Key " + element.getObjectKey() + " is not Serializable and cannot be replicated.");
            return;
        }
        if (!element.isSerializable()) {
            FineLoggerFactory.getLogger().error("Object with key " + element.getObjectKey() + " is not Serializable and cannot be updated via copy");
            return;
        }
        JGroupEventMessage e = new JGroupEventMessage(JGroupEventMessage.PUT, (Serializable) element.getObjectKey(), element,
                cache.getName(), this.asynchronousReplicationInterval);

        sendNotification(cache, e);
    }
```
封装一个JGroupEventMessage.PUT事件进行发送，接下来看一下其怎么发送，发送给谁。
```
protected void sendNotification(Ehcache cache, JGroupEventMessage eventMessage) {
        final List<CachePeer> peers = this.listRemoteCachePeers(cache);

        for (final CachePeer peer : peers) {
            try {
                peer.send(Collections.singletonList(eventMessage));
            } catch (RemoteException e) {
                FineLoggerFactory.getLogger().error("Failed to send message '" + eventMessage + "' to peer '" + peer + "'", e);
            }
        }

    }
```
获取到节点列表CachePeer，直接看CachePeer是怎么初始化以及更新时机
```
private List<CachePeer> listRemoteCachePeers(Ehcache cache) {
        final CacheManager cacheManager = cache.getCacheManager();
        final CacheManagerPeerProvider provider = cacheManager.getCacheManagerPeerProvider(JGroupsCacheManagerPeerProvider.SCHEME_NAME);
        return provider.listRemoteCachePeers(cache);
    }
```
直接从CacheManager里面获取CacheManagerPeerProvider让CacheManagerPeerProvider进行提供。
这里的CacheManagerPeerProvider就是一开始初始化提到的JGroupsCacheManagerPeerProvider
所以直接看其listRemoteCachePeers方法。
```
public List<CachePeer> listRemoteCachePeers(Ehcache cache) throws CacheException {
        if (this.cachePeersListCache == null) {
            return Collections.emptyList();
        }
        return this.cachePeersListCache;
    }
```
只是简单的返回cachePeersListCache
同时注意到，在JGroupsCacheManagerPeerProvider类的init方法中有这么一段
```
this.cachePeer = new JGroupsCachePeer(this.channel, clusterName);
...
this.cachePeersListCache = Collections.singletonList((CachePeer) this.cachePeer);
```
所以可以看出这边peer列表其实就是一个JGroupsCachePeer类，同时结合初始化中说JGroupsCacheReceiver作为消息接收类可以知道这两个类被分别用作写读类。
知道怎么发送添加信息之后接下来看JGroupsCacheReceiver接收到添加元素消息之后怎么进行处理。
直接看receive(Message msg)方法。
```
 if (object instanceof JGroupEventMessage) {
            this.safeHandleJGroupNotification((JGroupEventMessage) object);
}
```

safeHandleJGroupNotification=>handleJGroupNotification=>handleCacheManagerNotification=>handleEhcacheNotification
```
private void handleEhcacheNotification(final JGroupEventMessage message) {
        final String cacheName = message.getCacheName();
        final Ehcache cache = this.cacheManager.getEhcache(cacheName);
        if (cache == null) {
            FineLoggerFactory.getLogger().error("Received message " + message + " for cache that does not exist: " + cacheName);
            return;
        }

        switch (message.getEvent()) {
            case JGroupEventMessage.REMOVE_ALL: {
                FineLoggerFactory.getLogger().debug("received remove all:      cache={}", cacheName);
                cache.removeAll(true);
                break;
            }
            case JGroupEventMessage.REMOVE: {
                final Serializable serializableKey = message.getSerializableKey();
                cache.remove(serializableKey, true);
                break;
            }
            case JGroupEventMessage.PUT: {
                final Serializable serializableKey = message.getSerializableKey();
                FineLoggerFactory.getLogger().debug("received put:             cache={}, key={}", cacheName, serializableKey);
                cache.put(message.getElement(), true);
                break;
            }

            default: {
                FineLoggerFactory.getLogger().error("Unknown JGroupsEventMessage type received, ignoring message: " + message);
                break;
            }
        }
    }
```
可以看到增删都是直接操作本地Ehcache

但是这边有一个问题，那就是修改的事件怎么没有。同理直接看
JGroupsCacheReplicator.notifyElementUpdated
即
```
public void notifyElementUpdated(Ehcache cache, Element element) throws CacheException {
        if (notAlive() || !replicateUpdates) {
            return;
        }

        if (replicateUpdatesViaCopy) {
            replicatePutNotification(cache, element);
        } else {
            replicateRemoveNotification(cache, element);
        }

    }
```
如果配置了replicateUpdatesViaCopy其实就是一个put操作
如果没有配置，直接通知别的节点直接删除该元素。
目前配置的是第一种情况，对于第二中配置目前还暂未支持。


## 问题

1: ehcache的put和remove的时候是否需要集群锁来保障。即put和remove的操作是否需要集群原子性来保障。

因为目前

2：update方法是否有必要。


