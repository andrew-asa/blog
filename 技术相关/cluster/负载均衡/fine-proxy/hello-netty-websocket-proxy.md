# 目的
在hello-netty-websocket中已经实现了一个最简单的websocket客户端和服务启动端。
但是如果我想在客户端和服务器端不改边任何代码逻辑然后再加一个中间层，使得所有经过websocket服务器端的流量都先经过这个中间层，所有返回客户端的流量也都先经过这个中间层，这就是websocket代理。
具体项目代码见：
https://cloud.finedevelop.com/projects/RS/repos/ws-proxy/browse

大体流程即
```
websocket client <==> websocket proxy  <==> websocket server
```

另websocket proxy监听的端口为38889
websocket server监听的端口为8083
这时候就是 
客户端发送
ws://localhost:38889/websocket
代理服务端监到该请求然后进行转发到
ws://localhost:8083/websocket

具体代码
1: 代理服务器WebSocketProxyServer
```
package com.asa.lab.ws.proxy.server;

import com.corundumstudio.socketio.SocketIOChannelInitializer;
import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.Channel;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.http.HttpObjectAggregator;
import io.netty.handler.codec.http.HttpServerCodec;
import io.netty.handler.stream.ChunkedWriteHandler;

/**
 * @author andrew_asa
 * @date 2019-09-16.
 */
public class WebSocketProxyServer {

    public void run(int port) throws Exception {

        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            SocketIOChannelInitializer pipelineFactory = new SocketIOChannelInitializer();
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    //.childHandler(pipelineFactory);
                    .childHandler(new ChannelInitializer<SocketChannel>() {

                        @Override
                        protected void initChannel(SocketChannel ch)
                                throws Exception {

                            ChannelPipeline pipeline = ch.pipeline();
                            pipeline.addLast("http-codec",
                                             new HttpServerCodec());
                            pipeline.addLast("aggregator",
                                             new HttpObjectAggregator(65536));
                            ch.pipeline().addLast("http-chunked",
                                                  new ChunkedWriteHandler());
                            pipeline.addLast("WebSocketServerHandler",
                                             new WebSocketServerHandler());
                        }
                    });

            Channel ch = b.bind(port).sync().channel();
            System.out.println("Web socket server started at port " + port
                                       + '.');
            System.out
                    .println("Open your browser and navigate to http://localhost:"
                                     + port + '/');

            ch.closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }

    public static void main(String[] args) throws Exception {

        int port = 38889;
        if (args.length > 0) {
            try {
                port = Integer.parseInt(args[0]);
            } catch (NumberFormatException e) {
                e.printStackTrace();
            }
        }
        new WebSocketProxyServer().run(port);
    }
}
```
监听处理类WebSocketProxyServerHandler
```
package com.asa.lab.ws.proxy.server;

import com.asa.lab.ws.proxy.client.WebsocketProxyClient;
import com.asa.lab.ws.proxy.client.WebsocketProxyClientCenter;
import com.asa.log.LoggerFactory;
import io.netty.buffer.ByteBufOutputStream;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelFutureListener;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.handler.codec.http.DefaultFullHttpResponse;
import io.netty.handler.codec.http.FullHttpRequest;
import io.netty.handler.codec.http.FullHttpResponse;
import io.netty.handler.codec.http.HttpHeaders;
import io.netty.handler.codec.http.HttpMethod;
import io.netty.handler.codec.http.HttpResponseStatus;
import io.netty.handler.codec.http.HttpUtil;
import io.netty.handler.codec.http.websocketx.WebSocketFrame;
import io.netty.handler.codec.http.websocketx.WebSocketServerHandshaker;
import org.apache.http.Header;
import org.apache.http.HttpEntity;
import org.apache.http.HttpResponse;
import org.apache.http.client.HttpClient;
import org.apache.http.client.methods.HttpDelete;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.client.methods.HttpHead;
import org.apache.http.client.methods.HttpPatch;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.client.methods.HttpPut;
import org.apache.http.client.methods.HttpRequestBase;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.impl.conn.PoolingHttpClientConnectionManager;
import org.apache.http.util.EntityUtils;

import java.net.URI;
import java.util.Set;
import java.util.logging.Logger;

import static io.netty.handler.codec.http.HttpVersion.HTTP_1_1;
import static io.netty.handler.codec.rtsp.RtspResponseStatuses.BAD_REQUEST;

/**
 * @author andrew_asa
 * @date 2019-09-16.
 */
public class WebSocketProxyServerHandler extends SimpleChannelInboundHandler<Object> {

    private static final Logger logger = Logger
            .getLogger(WebSocketProxyServerHandler.class.getName());


    private WebSocketServerHandshaker handshaker;

    private WebsocketProxyClient websocketClient;


    private WebsocketProxyClientCenter clientCenter;

    public static final String NAME = "WebSocketProxyServerHandler";

    public WebSocketProxyServerHandler() {

        clientCenter = new WebsocketProxyClientCenter();
    }

    @Override
    public void channelRead0(ChannelHandlerContext ctx, Object msg)
            throws Exception {
        // 传统的HTTP接入
        if (msg instanceof FullHttpRequest) {
            handleHttpRequest(ctx, (FullHttpRequest) msg);
        }
        // WebSocket接入
        else if (msg instanceof WebSocketFrame) {
            LoggerFactory.getLogger().info("------ websocket ----");
            handleWebSocketFrame(ctx, (WebSocketFrame) msg);
        }
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {

        ctx.flush();
    }

    private void handleHttpRequest(ChannelHandlerContext ctx,
                                   FullHttpRequest req) throws Exception {

        // 如果HTTP解码失败，返回HHTP异常
        if (!req.decoderResult().isSuccess()
                || (!"websocket".equals(req.headers().get("Upgrade")))) {
            sendHttpResponse(ctx, req, new DefaultFullHttpResponse(HTTP_1_1,
                                                                   BAD_REQUEST));
            return;
        }
        //
        //if (!req.getUri().startsWith("ws")) {
        //    // http 转发
        //    //
        //    return;
        //}
        WebsocketProxyClient proxyClient = clientCenter.newWebsocketProxyClient(ctx, req);
        proxyClient.connect(ctx, req);
    }

    private void handleWebSocketFrame(ChannelHandlerContext ctx,
                                      WebSocketFrame frame) {

        WebsocketProxyClient proxyClient = clientCenter.getWebsocketProxyClient(ctx);
        proxyClient.clientChannelRead(ctx, frame);
    }

    private void sendHttpResponse(ChannelHandlerContext ctx,
                                  FullHttpRequest req, FullHttpResponse res) {

        String dest = getDest(req);
        HttpClient httpClient = HttpClients.custom().setConnectionManager(new PoolingHttpClientConnectionManager()).build();
        HttpMethod method = req.method();
        HttpRequestBase httpRequestBase = createHttpRequestBase(method.name());

        HttpHeaders headers = req.headers();
        Set<String> names = headers.names();

        for (String name : names) {
            if (!validHeaderName(name)) {
                httpRequestBase.addHeader(name, headers.get(name));
            }
        }

        // url 请求参数
        //req.
        try {
            logger.info(req.uri() + "->" + dest);
            httpRequestBase.setURI(new URI(dest));
            HttpResponse httpResponse = httpClient.execute(httpRequestBase);
            res.setStatus(HttpResponseStatus.valueOf(httpResponse.getStatusLine().getStatusCode()));
            for (Header header : httpResponse.getAllHeaders()) {
                res.headers().add(header.getName(), header.getValue());
            }
            HttpEntity resEntity = httpResponse.getEntity();
            ByteBufOutputStream out = new ByteBufOutputStream(res.content());
            if (resEntity != null) {
                try {
                    resEntity.writeTo(out);
                } finally {
                    //res.flushBuffer();
                    EntityUtils.consume(resEntity);
                }
                HttpUtil.setContentLength(res, res.content().readableBytes());
            }
        } catch (Exception e) {
            e.printStackTrace();
        }

        // 返回应答给客户端
        //if (res.getStatus().code() != 200) {
        //    ByteBuf buf = Unpooled.copiedBuffer(res.getStatus().toString(),
        //                                        CharsetUtil.UTF_8);
        //    res.content().writeBytes(buf);
        //    buf.release();
        //    HttpUtil.setContentLength(res, res.content().readableBytes());
        //}

        // 如果是非Keep-Alive，关闭连接
        ChannelFuture f = ctx.channel().writeAndFlush(res);
        if (!HttpUtil.isKeepAlive(req) || res.status().code() != 200) {
            f.addListener(ChannelFutureListener.CLOSE);
        }
    }

    public HttpRequestBase createHttpRequestBase(String method) {

        if (method.equalsIgnoreCase("get")) {
            return new HttpGet();
        }
        if (method.equalsIgnoreCase("put")) {
            return new HttpPut();
        }
        if (method.equalsIgnoreCase("delete")) {
            return new HttpDelete();
        }
        if (method.equalsIgnoreCase("patch")) {
            return new HttpPatch();
        }
        if (method.equalsIgnoreCase("head")) {
            return new HttpHead();
        }
        return new HttpPost();
    }


    private boolean validHeaderName(String headerName) {

        return headerName.equalsIgnoreCase("content-length");
    }

    private String getDest(FullHttpRequest req) {

        return "http://localhost:38888" + req.uri();
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
            throws Exception {

        cause.printStackTrace();
        ctx.close();
    }
}

```
1: 其实和hello-netty-websocket中的websocket服务器端处理差不多，不过就是变成了WebsocketProxyClient对象的connet方法
```
package com.asa.lab.ws.proxy.client;

import com.asa.log.LoggerFactory;
import io.netty.bootstrap.Bootstrap;
import io.netty.channel.Channel;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.http.FullHttpRequest;
import io.netty.handler.codec.http.HttpClientCodec;
import io.netty.handler.codec.http.HttpHeaders;
import io.netty.handler.codec.http.HttpObjectAggregator;
import io.netty.handler.codec.http.websocketx.TextWebSocketFrame;
import io.netty.handler.codec.http.websocketx.WebSocketClientHandshakerFactory;
import io.netty.handler.codec.http.websocketx.WebSocketFrame;
import io.netty.handler.codec.http.websocketx.WebSocketServerHandshaker;
import io.netty.handler.codec.http.websocketx.WebSocketServerHandshakerFactory;
import io.netty.handler.codec.http.websocketx.WebSocketVersion;

import java.net.URI;

/**
 * @author andrew_asa
 * @date 2019-09-17.
 */
public class WebsocketProxyClient {

    private String url;

    private URI uri;


    private WebSocketServerHandshaker clientHandshaker;

    /**
     * 客户端channel
     */
    private Channel clientChannel;

    private Channel serverChannel;

    private EventLoopGroup group = new NioEventLoopGroup();

    public WebsocketProxyClient(ChannelHandlerContext ctx, FullHttpRequest req) {

        String uri = req.uri();
        url = "ws://localhost:8083" + uri;
    }

    /**
     * 客户端建立连接
     */
    public void connect(ChannelHandlerContext ctx, FullHttpRequest req) {

        clientChannel = ctx.channel();
        // 构造握手响应返回，本机测试
        WebSocketServerHandshakerFactory wsFactory = new WebSocketServerHandshakerFactory(url, null, false);
        clientHandshaker = wsFactory.newHandshaker(req);
        if (clientHandshaker == null) {
            WebSocketServerHandshakerFactory
                    .sendUnsupportedVersionResponse(ctx.channel());
        } else {
            uri = URI.create(url);
            // 协议升级
            clientHandshaker.handshake(ctx.channel(), req);
            // 连接服务器端
            Bootstrap b = new Bootstrap();
            String protocol = uri.getScheme();
            final WebSocketProxyClientHandler handler =
                    new WebSocketProxyClientHandler(
                            WebSocketClientHandshakerFactory.newHandshaker(
                                    uri, WebSocketVersion.V13, null, false, HttpHeaders.EMPTY_HEADERS, 1280000), clientChannel);
            try {
                b.group(group)
                        .channel(NioSocketChannel.class)
                        .handler(new ChannelInitializer<SocketChannel>() {

                            @Override
                            public void initChannel(SocketChannel ch) throws Exception {

                                ChannelPipeline pipeline = ch.pipeline();
                                pipeline.addLast("http-codec", new HttpClientCodec());
                                pipeline.addLast("aggregator", new HttpObjectAggregator(65536));
                                pipeline.addLast(WebSocketProxyClientHandler.NAME, handler);
                            }
                        });

                serverChannel = b.connect(uri.getHost(), uri.getPort()).sync().channel();
                handler.handshakeFuture().sync();
                LoggerFactory.getLogger().info("connect to {} ", url);
            } catch (Exception e) {
                LoggerFactory.getLogger().error(e.getMessage(), e);
            }
        }
    }

    /**
     * 客户端的channel可读
     *
     * @param ctx
     * @param frame
     */
    public void clientChannelRead(ChannelHandlerContext ctx, WebSocketFrame frame) {

        if (frame instanceof TextWebSocketFrame) {
            TextWebSocketFrame msg = new TextWebSocketFrame(((TextWebSocketFrame) frame).text());
            // 往服务器发送
            serverChannel.writeAndFlush(msg);
        }
    }
}

```
可以看到
1.1 也是先和客户端进行握手升级
1.2 把自己变成一个websocket客户端。先叫做代理客户端。
先看代理客户端的监听处理类。WebSocketProxyClientHandler
```
package com.asa.lab.ws.proxy.client;

import com.asa.log.LoggerFactory;
import io.netty.channel.Channel;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelPromise;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.handler.codec.http.FullHttpResponse;
import io.netty.handler.codec.http.websocketx.TextWebSocketFrame;
import io.netty.handler.codec.http.websocketx.WebSocketClientHandshaker;
import io.netty.util.CharsetUtil;


/**
 * @author andrew_asa
 * @date 2019-09-17.
 */
public class WebSocketProxyClientHandler extends SimpleChannelInboundHandler<Object> {

    WebSocketClientHandshaker handshaker;

    ChannelPromise handshakeFuture;

    private WebsocketProxyClient client;

    private Channel clientChannel;

    public static final String NAME = "WebSocketProxyClientHandler";

    public WebSocketProxyClientHandler(final WebSocketClientHandshaker handshaker, Channel clientChannel) {

        this.handshaker = handshaker;
        this.clientChannel = clientChannel;
    }

    public ChannelFuture handshakeFuture() {

        return handshakeFuture;
    }

    @Override
    public void handlerAdded(final ChannelHandlerContext ctx) throws Exception {

        handshakeFuture = ctx.newPromise();
    }

    @Override
    public void channelActive(final ChannelHandlerContext ctx) throws Exception {

        handshaker.handshake(ctx.channel());
    }

    @Override
    public void channelInactive(final ChannelHandlerContext ctx) throws Exception {

        System.out.println("WebSocket Client disconnected!");
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, Object msg) throws Exception {

        final Channel ch = ctx.channel();
        if (!handshaker.isHandshakeComplete()) {
            // web socket client connected
            LoggerFactory.getLogger().info("HandshakeComplete");
            handshaker.finishHandshake(ch, (FullHttpResponse) msg);
            handshakeFuture.setSuccess();
            return;
        }
        LoggerFactory.getLogger().info("channelRead0 from {},msg {}", ch, msg);
        if (msg instanceof FullHttpResponse) {
            final FullHttpResponse response = (FullHttpResponse) msg;
            throw new Exception("Unexpected FullHttpResponse (getStatus=" + response.getStatus() + ", content="
                                        + response.content().toString(CharsetUtil.UTF_8) + ')');
        }
        // 直接像客户端发送即可
        if (msg instanceof TextWebSocketFrame) {
            TextWebSocketFrame send = new TextWebSocketFrame(((TextWebSocketFrame) msg).text());
            LoggerFactory.getLogger().info(" msg {}  from {} to {}", ((TextWebSocketFrame) msg).text(), ch, clientChannel);
            // 往服务器发送
            clientChannel.writeAndFlush(send);
        }
    }

    @Override
    public void exceptionCaught(final ChannelHandlerContext ctx, final Throwable cause) throws Exception {

        cause.printStackTrace();

        if (!handshakeFuture.isDone()) {
            handshakeFuture.setFailure(cause);
        }

        ctx.close();
    }
}
```
和一般的websocket处理没多大差别只是在收到消息服务端（8083）发来的消息时候直接向真正的客户端进行发送即可。

2：如果代理服务器端收到的是websocket客户端发过来的websocket消息
```
 serverChannel.writeAndFlush(msg);
```
直接往真正的websocket服务器进行发送。

## 最后

