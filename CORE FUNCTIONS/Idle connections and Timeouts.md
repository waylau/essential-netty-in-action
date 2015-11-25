空闲连接以及超时
====

检测空闲连接和超时是为了及时释放资源。常见的方法发送消息用于测试一个不活跃的连接来,通常称为“心跳”,到远端来确定它是否还活着。(一个更激进的方法是简单地断开那些指定的时间间隔的不活跃的连接)。

处理空闲连接是一项常见的任务,Netty 提供了几个 ChannelHandler 实现此目的。表8.4概述。

Table 8.4 ChannelHandlers for idle connections and timeouts

名称 | 描述
-----|----
IdleStateHandler | 如果连接闲置时间过长，则会触发 IdleStateEvent 事件。在 ChannelInboundHandler 中可以覆盖  userEventTriggered(...) 方法来处理 IdleStateEvent。
ReadTimeoutHandler | 在指定的时间间隔内没有接收到入站数据则会抛出 ReadTimeoutException 并关闭 Channel。ReadTimeoutException 可以通过覆盖 ChannelHandler 的 exceptionCaught(…) 方法检测到。
WriteTimeoutHandler | WriteTimeoutException 可以通过覆盖 ChannelHandler 的 exceptionCaught(…) 方法检测到。

详细看下 IdleStateHandler，下面是一个例子，当超过60秒没有数据收到时，就会得到通知，此时就发送心跳到远端，如果没有回应，连接就关闭。

Listing 8.7 Sending heartbeats

	public class IdleStateHandlerInitializer extends ChannelInitializer<Channel> {
	
	    @Override
	    protected void initChannel(Channel ch) throws Exception {
	        ChannelPipeline pipeline = ch.pipeline();
	        pipeline.addLast(new IdleStateHandler(0, 0, 60, TimeUnit.SECONDS));  //1
	        pipeline.addLast(new HeartbeatHandler());
	    }
	
	    public static final class HeartbeatHandler extends ChannelInboundHandlerAdapter {
	        private static final ByteBuf HEARTBEAT_SEQUENCE = Unpooled.unreleasableBuffer(
	                Unpooled.copiedBuffer("HEARTBEAT", CharsetUtil.ISO_8859_1));  //2
	
	        @Override
	        public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
	            if (evt instanceof IdleStateEvent) {
	                 ctx.writeAndFlush(HEARTBEAT_SEQUENCE.duplicate())
	                         .addListener(ChannelFutureListener.CLOSE_ON_FAILURE);  //3
	            } else {
	                super.userEventTriggered(ctx, evt);  //4
	            }
	        }
	    }
	}

1. IdleStateHandler 将通过 IdleStateEvent 调用 userEventTriggered ，如果连接没有接收或发送数据超过60秒钟
2. 心跳发送到远端
3. 发送的心跳并添加一个侦听器，如果发送操作失败将关闭连接
4. 事件不是一个 IdleStateEvent 的话，就将它传递给下一个处理程序

总而言之,这个例子说明了如何使用 IdleStateHandler 测试远端是否还活着，如果不是就关闭连接释放资源。