# 
```
/** 
 *  io事件状态改变的回调方法，用户可以实现接口来得到状态改变通知 
 */  
public interface ChannelInboundHandler extends ChannelHandler {  
  
    /** 
     * 当Channel注册到EventLoop时触发（只会触发一次） 
     * register事件：即 在jdk的 selector上注册channel ， javaChannel().register(eventLoop().selector, 0, this(NioSocketChannel)); 
     * 注册完成后触发channelRegistered 
     * 事件源：Bootstrap　b; b.connect(serverIp, port)时异步触发register,register完成后，触发channelRegistered 
     *  
     */  
    void channelRegistered(ChannelHandlerContext ctx) throws Exception;  
  
    /** 
     * 当Channel从EventLoop注销时触发 
     */  
    void channelUnregistered(ChannelHandlerContext ctx) throws Exception;  
  
    /** 
     * 当Channel是活动时触发 
     * 活动：网络connect ==true 
     */  
    void channelActive(ChannelHandlerContext ctx) throws Exception;  
  
    /** 
     *  
     * 当已注册的Channel，不再活动，已到生命周期结束时触发 
     */  
    void channelInactive(ChannelHandlerContext ctx) throws Exception;  
  
    /** 
     × 
     *　当Channel从对端接收到数据时触发 
     */  
    void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception;  
  
    /** 
     * Invoked when the last message read by the current read operation has been consumed by 
     * {@link #channelRead(ChannelHandlerContext, Object)}.  If {@link ChannelOption#AUTO_READ} is off, no further 
     * attempt to read an inbound data from the current {@link Channel} will be made until 
     * {@link ChannelHandlerContext#read()} is called. 
     *　 
     */  
    void channelReadComplete(ChannelHandlerContext ctx) throws Exception;  
  
    /** 
     * 
     *　当用户事件被触发时调用 
     */  
    void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception;  
  
    /** 
     * 
     *　当channel的可写入状态改变时触发，可以通过 Channel.isWritable()检查是否可以写入 
     */  
    void channelWritabilityChanged(ChannelHandlerContext ctx) throws Exception;  
  
    /** 
     * 
     *　当前生异常时触发 
     */  
    @Override  
    @SuppressWarnings("deprecation")  
    void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception;  
}  
/** 
 * 
 * 一个抽像的类实现了ChannelInboundHandler的所有方法， 
 * 实现仅仅是把事件转发给下一个ChannelHandler，即本类不处理任务事件，可以在子类中 
 * 覆盖方法实现逻辑。目的：子类可以覆盖需要的方法，其它方法采用默认实现--简单。 
 * </p> 
 */  
public class ChannelInboundHandlerAdapter extends ChannelHandlerAdapter implements ChannelInboundHandler {  
  
  
    public void channelRegistered(ChannelHandlerContext ctx) throws Exception {  
        ctx.fireChannelRegistered();  
    }  
    public void channelUnregistered(ChannelHandlerContext ctx) throws Exception {  
        ctx.fireChannelUnregistered();  
    }  
    public void channelActive(ChannelHandlerContext ctx) throws Exception {  
        ctx.fireChannelActive();  
    }  
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {  
        ctx.fireChannelInactive();  
    }  
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {  
        ctx.fireChannelRead(msg);  
    }  
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {  
        ctx.fireChannelReadComplete();  
    }  
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {  
        ctx.fireUserEventTriggered(evt);  
    }  
    public void channelWritabilityChanged(ChannelHandlerContext ctx) throws Exception {  
        ctx.fireChannelWritabilityChanged();  
    }  
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)  
            throws Exception {  
        ctx.fireExceptionCaught(cause);  
    }  
}  
  
  
  
/** 
 *  
 * 一个特殊的ChannelInboundHandler实现，目的：提供一个简单的方法，当Channel被注册到EventLoop时触发，用于初始化Channel 
 * 通常用于Bootstrap#handler(ChannelHandler)、ServerBootstrap#handler(ChannelHandler)、ServerBootstrap#childHandler(ChannelHandler) 
 * 用于设置Channel的ChannelPipeline，添加子ChannelHandler. 
 * 子类需要实现initChannel方法 
 *  
 *  
 * 示例 ： 
 * public class MyChannelInitializer extends {@link ChannelInitializer} { 
 *     public void initChannel({@link Channel} channel) { 
 *         channel.pipeline().addLast("myHandler", new MyHandler()); 
 *     } 
 * } 
 *  ServerBootstrap  serverBootstrap = ...; 
 * serverBootstrap.childHandler(new MyChannelInitializer()); 
 */  
@Sharable  
public abstract class ChannelInitializer<C extends Channel> extends ChannelInboundHandlerAdapter {  
  
   
    /** 
     * 当Channel被注册到EventLoop时触发，initChannel方法中需要对Channel初始化，这时Channel.pipeline中只有一个 
     * ChannelInitializer（Bootstrap#handler(ChannelInitializer)时添加）， 
     * initChannel方法执行完程后--（已经添加了N个ChannelHandler），自动的从Channel.pipeline移除这个ChannelHandler, 
     *  原因：初始化对于channel只需要调用一次 
     */  
    protected abstract void initChannel(C ch) throws Exception;  
      
    /* 
    * ignore other methods 
    */  
}  
```