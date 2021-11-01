# 传输协议相关
+ 具体到发送给某个节点sendUnicast


## 在jgroup-CENTRAL_LOCK 提到一个关于锁的关键问题，即如果不能保证点到点的消息的正确交付，如何保证上层应用的锁的正确。在目前的配置中是

即CENTRAL_LOCK 发送的消息都是
sendLockResponse(rsp.type, rsp.owner, rsp.lock_name, rsp.lock_id);
这里发送的最终会调用down_prot.down(new Event(Event.MSG, msg));
这里的down_port 指的是在xml配置在CENTRAL_LOCK上面的协议。
即这里会调用
UDP 或者 TCP 的down方法。所以有必要分析一下UDP 或者 TCP 的down方法。
又因为UDP 或者 TCP都是可以是继承TP，所以先看一下TP的down方法。
直接上源码，省略不必要的代码。
```
public Object down(Event evt) {
        ...
        try {
            send(msg, dest, multicast);
        }
        catch(InterruptedIOException iex) {
        }
        ...
        return null;
    }
```
在真正的源码中处理的情况比这个复杂，有对消息的复制，回环路，本地交付给本地地址的处理。但这些都是次要的条件，继续看send方法。

```
protected void send(Message msg, Address dest, boolean multicast) throws Exception {
        ...
        ExposedByteArrayOutputStream out_stream=new ExposedByteArrayOutputStream((int)(msg.size() + 50));
        ExposedDataOutputStream dos=new ExposedDataOutputStream(out_stream);
        writeMessage(msg, dos, multicast);
        Buffer buf=new Buffer(out_stream.getRawBuffer(), 0, out_stream.size());
        doSend(buf, dest, multicast);
     ...
    }
```
消息复制，然后真正调用doSend方法。继续看doSend方法。
```
protected void doSend(Buffer buf, Address dest, boolean multicast) throws Exception {
        ...
        if(multicast)
            sendMulticast(buf.getBuf(), buf.getOffset(), buf.getLength());
        else
            sendToSingleMember(dest, buf.getBuf(), buf.getOffset(), buf.getLength());
    }
```
这边判断是否是多播，可以认为如果不指定具体地址就调用sendMulticast方法，但是在协议者向客户端发送被授权请求走的是sendToSingleMember。所以继续看
sendToSingleMember方法。

```
protected void sendToSingleMember(Address dest, byte[] buf, int offset, int length) throws Exception {
        ...
        if(physical_dest != null)
            sendUnicast(physical_dest, buf, offset, length);
        else if(log.isWarnEnabled())
            log.warn(Util.getMessage("PhysicalAddrMissing"), local_addr, dest);
    }
```
可以看到如果是物理地址不为null则调用sendUnicast方法。继续看sendUnicast方法。
sendUnicast是一个接口，udp和tcp协议有不同的实现，先看一下udp协议的实现。
```
 public void sendUnicast(PhysicalAddress dest, byte[] data, int offset, int length) throws Exception {
        _send(((IpAddress)dest).getIpAddress(), ((IpAddress)dest).getPort(), false, data, offset, length);
    }
```
继续
```
protected void _send(InetAddress dest, int port, boolean mcast, byte[] data, int offset, int length) throws Exception {
        DatagramPacket packet=new DatagramPacket(data, offset, length, dest, port);
        try {
            if(mcast) {
                if(mcast_sock != null && !mcast_sock.isClosed()) {
                    try {
                        mcast_sock.send(packet);
                    }
                    catch(NoRouteToHostException e) {
                        mcast_sock.setInterface(mcast_sock.getInterface());
                    }
                }
            }
            else {
                if(sock != null && !sock.isClosed())
                    sock.send(packet);
            }
        }
        catch(Exception ex) {
            throw new Exception("dest=" + dest + ":" + port + " (" + length + " bytes)", ex);
        }
    }
```
先不管mcast参数是什么先看mcast_sock，sock这两个对象
+ sock ==> DatagramSocket udp 的套接字。
+ mcast_sock ==> MulticastSocket 多播套接字。
udp 的可靠性到此证明完毕，再看tcp的sendUnicast方法。
由于tcp 继承BasicTCP所以直接看BasicTCP的sendUnicast方法。

```
 public void sendUnicast(PhysicalAddress dest, byte[] data, int offset, int length) throws Exception {
        if(log.isTraceEnabled()) log.trace("dest=" + dest + " (" + length + " bytes)");
        send(dest, data, offset, length);
    }
```
直接看Tcp的send方法。
```
public void send(Address dest, byte[] data, int offset, int length) throws Exception {
        if(ct != null)
            ct.send(dest, data, offset, length);
    }
```
ct 为TCPConnectionMap 直接看起send方法。
```
public void send(Address dest, byte[] data, int offset, int length) throws Exception {  
        ....      
        TCPConnection conn=null;
        try {
            conn=mapper.getConnection(dest);
        }
        catch(Throwable t) {
        }
        ....
        if(conn != null) {
            try {
                conn.send(data, offset, length);
            }
            catch(Exception ex) {
                mapper.removeConnectionIfPresent(dest,conn);
                throw ex;
            }
        }
    }
```

上面省略了一些代码。直接看TCPConnection的send方法
```
        protected void send(byte[] data, int offset, int length) throws Exception {
            if (sender != null) {
                sender.addToQueue(tmp);
            }
            else
                _send(data, offset, length, true);
        }
```
直接看_send方法。
```
 protected void _send(byte[] data, int offset, int length, boolean acquire_lock) throws Exception {
            if(acquire_lock)
                send_lock.lock();
            try {
                doSend(data, offset, length);
                updateLastAccessed();
            }
            catch(InterruptedException iex) {
                Thread.currentThread().interrupt(); // set interrupt flag again
            }        
            finally {
                if(acquire_lock)
                    send_lock.unlock();
            }
        }

        protected void doSend(byte[] data, int offset, int length) throws Exception {
            out.writeInt(length); // write the length of the data buffer first
            out.write(data,offset,length);
            out.flush(); // may not be very efficient (but safe)           
        }
```
可以看到这边最终会调用out.write。先看一下这个out怎么来的。
```
public TCPConnection(Socket s) throws Exception {
            if(s == null)
                throw new IllegalArgumentException("Invalid parameter s=" + s);                       
            setSocketParameters(s);
            this.out=new DataOutputStream(new BufferedOutputStream(s.getOutputStream()));
            this.in=new DataInputStream(new BufferedInputStream(s.getInputStream()));
            this.peer_addr=readPeerAddress(s);            
            this.sock=s;
        }
```
构造的时候得来的。可以看出这是一个Socket，tcp的socket。

通过上面的代码可以证明两点在节点之间通讯的时候。
1: jgroup tcp 协议最终用底层的Socket发送消息
2：jgroup upd 协议最终用底层的DatagramSocket发送消息。
所以
```
<UDP/>
<CENTRAL_LOCK num_backups="2"/>
其它协议
```
配置是不可取的。那是否是说用jgroup 就不应该用udp 当然不是，因为有UNICAST3等协议，具体解见jgroup-UNICAST所以正确的配置应该是
```
    <TCP />
    ...
    <UNICAST3/>
    ...
    <CENTRAL_LOCK num_backups="2"/>
```
确保在CENTRAL_LOCK消息交付之间有UNICAST3保障。

## TCP
### 参数
参数| 作用 | 默认值
--- | ---|---
start_port| 本节点监听的端口如设置为0,将会随机选定一个端口绑定 | 7800 
end_port | 尝试监听的最大端口，如果在start_port和end_port指定的范围内都找不到一个可用端口，则抛出一个BindException。如果不指定end_port，或者end_port<start_port，end_port将没有上限，如果start_port=end_port，JGroup将会只用该端口来绑定。 | 
bind_addr | 监听绑定的地址JGroup监听的物理IP，如不指定，JGroup将会绑定到所有可用网络接口。当有多个网络接口时应该指定一个 | 


## UDP
+ 配置
+ mcast_addr 多播地址
+ mcast_port 多播端口






## 参考链接
[使用MulticastSocket实现多点广播 https://blog.csdn.net/jiangxinyu/article/details/8158329#](https://blog.csdn.net/jiangxinyu/article/details/8158329#)