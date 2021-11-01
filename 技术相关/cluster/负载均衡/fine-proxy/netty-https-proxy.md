# https 代理
1：证书和私钥的制作参考https://kms.finedevelop.com/pages/viewpage.action?pageId=49528638
2：证书和私钥的读取
证书的读取
```
CertificateFactory cf = CertificateFactory.getInstance("X.509");
        return (X509Certificate) cf.generateCertificate(inputStream);
```
这里的inputStream就是证书文件的输入流
私钥的读取
```
byte[] bts = ...
EncodedKeySpec privateKeySpec = new PKCS8EncodedKeySpec(bts);
        PrivateKey pk= getKeyFactory().generatePrivate(privateKeySpec);
```
这里的bts就是私钥的字节数组
注意采用
`openssl genrsa -des3 -out server.key 2048`
`openssl rsa -in server.key -out server.key`
生成的私钥并不能被java正确读取，所以需要对其进行转换一下，命令如下
`openssl pkcs8 -topk8 -inform PEM -outform DER -in server.key -nocrypt > pkcs8_key.key`
pkcs8_key.key就是被上面读取的字节数组。

netty 操作https
直接上最主要的handler
```
public class HttpsProxyServerHandle extends ChannelInboundHandlerAdapter {


    public static final String NAME = "HttpsProxyServerHandle";


    /**
     * http代理隧道握手成功
     */
    public final static HttpResponseStatus SUCCESS = new HttpResponseStatus(200,
                                                                            "Connection established");


    private HttpsProxyStrategy strategy;

    private ConfigureProvider configureProvider;

    private HttpsProxyConfigure httpsProxyConfigure;

    private ChannelFuture cf;

    private HttpsStatus status = HttpsStatus.START;

    private static final int SSH_HANDSHAKE = 22;

    enum HttpsStatus {
        START,
        CONNECT,
        CONNECTED
    }

    private List requestList;

    private boolean isConnect;

    public HttpsProxyServerHandle(ConfigureProvider configureProvider,
                                  HttpsProxyConfigure httpsProxyConfigure,
                                  HttpsProxyStrategy httpsProxyStrategy) {

        this.configureProvider = configureProvider;
        this.httpsProxyConfigure = httpsProxyConfigure;
        this.strategy = httpsProxyStrategy;
    }

    @Override
    public void channelRead(final ChannelHandlerContext ctx, final Object msg) throws Exception {

        if (msg instanceof HttpRequest) {
            HttpRequest request = (HttpRequest) msg;
            // 第一次建立连接取host和端口号和处理代理握手
            if (HttpsStatus.START.equals(status)) {
                status = HttpsStatus.CONNECT;
                if (HttpMethod.CONNECT.equals(request.method())) {
                    // 建立代理握手
                    status = HttpsStatus.CONNECTED;
                    HttpResponse response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, SUCCESS);
                    ctx.writeAndFlush(response);
                    ctx.channel().pipeline().remove("httpCodec");
                    ReferenceCountUtil.release(msg);
                    return;
                }
            }
            handleProxyData(ctx, ctx.channel(), request, true);
        } else if (msg instanceof HttpContent) {
            if (!HttpsStatus.CONNECTED.equals(status)) {
                handleProxyData(ctx, ctx.channel(), msg, true);
            } else {
                ReferenceCountUtil.release(msg);
                status = HttpsStatus.CONNECT;
            }
        } else {
            // ssl和websocket的握手处理
            ByteBuf byteBuf = (ByteBuf) msg;
            if (byteBuf.getByte(0) == SSH_HANDSHAKE) {
                // ssl握手
                SslContext sslCtx = SslContextBuilder
                        .forServer(httpsProxyConfigure.getPrivateKey(),
                                   httpsProxyConfigure.getX509Certificate())
                        .build();
                ctx.pipeline().addFirst("httpCodec", new HttpServerCodec());
                ctx.pipeline().addFirst("sslHandle", sslCtx.newHandler(ctx.alloc()));
                ctx.pipeline().fireChannelRead(msg);
                return;
            }
            handleProxyData(ctx, ctx.channel(), msg, false);
        }
    }

    @Override
    public void channelUnregistered(ChannelHandlerContext ctx) throws Exception {

        if (cf != null) {
            cf.channel().close();
        }
        ctx.channel().close();
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {

        if (cf != null) {
            cf.channel().close();
        }
        ctx.channel().close();
    }

    private void handleProxyData(ChannelHandlerContext ctx, Channel channel, Object msg, boolean isHttp) throws Exception {

        if (cf == null) {
            // connection异常 还有HttpContent进来，不转发
            if (isHttp && !(msg instanceof HttpRequest)) {
                return;
            }
            Bootstrap bootstrap = new Bootstrap();
            bootstrap.group(channel.eventLoop())
                    .channel(channel.getClass())
                    .handler(new HttpProxyInitializer(channel));
            requestList = new LinkedList();
            HttpsServer server = strategy.choice(ctx, null);
            cf = bootstrap.connect(server.getHost(), server.getPort());
            cf.addListener(new ChannelFutureListener() {

                @Override
                public void operationComplete(ChannelFuture future) throws Exception {

                    if (future.isSuccess()) {
                        future.channel().writeAndFlush(msg);
                        synchronized (requestList) {
                            for (Object obj : requestList) {
                                future.channel().writeAndFlush(obj);
                            }
                            requestList.clear();
                            isConnect = true;
                        }
                    } else {
                        // 这里还需要换另一个服务器来
                        for (Object obj : requestList) {
                            ReferenceCountUtil.release(obj);
                        }
                        requestList.clear();
                        future.channel().close();
                        channel.close();
                    }
                }
            });
        } else {
            synchronized (requestList) {
                if (isConnect) {
                    cf.channel().writeAndFlush(msg);
                } else {
                    requestList.add(msg);
                }
            }
        }
    }
}
```
最主要的是
```
// ssl和websocket的握手处理
            ByteBuf byteBuf = (ByteBuf) msg;
            if (byteBuf.getByte(0) == SSH_HANDSHAKE) {
                // ssl握手
                SslContext sslCtx = SslContextBuilder
                        .forServer(httpsProxyConfigure.getPrivateKey(),
                                   httpsProxyConfigure.getX509Certificate())
                        .build();
                ctx.pipeline().addFirst("httpCodec", new HttpServerCodec());
                ctx.pipeline().addFirst("sslHandle", sslCtx.newHandler(ctx.alloc()));
                ctx.pipeline().fireChannelRead(msg);
                return;
            }
```
这一段，主要是把证书返回给浏览器，然后后面剩下的加密解密的事情就是
sslCtx.newHandler的事情了，而https代理就可以像http代理那样得到征程的msg进行和后台服务器进行转发即可。




## 参考链接
[用Java读取使用证书](https://www.cnblogs.com/wen12128/archive/2010/10/11/1847995.html)
[SSL中，公钥、私钥、证书的后缀名都是些啥？](https://www.zhihu.com/question/29620953)
[深入理解HTTPS原理、过程与实践](https://zhuanlan.zhihu.com/p/26682342)
[浅谈HTTPS（SSL/TLS）原理](https://www.jianshu.com/p/41f7ae43e37b)
[集群配置HTTPS整理](https://kms.finedevelop.com/pages/viewpage.action?pageId=49528638)
[全站 HTTPS 的时代已经来了，你准备好了吗？](https://www.jianshu.com/p/6825d6c4bca6)
[RSA加密和数字签名在Java中常见应用](https://www.cnblogs.com/frankyou/p/11512027.html)
