实现
====

SPDY 使用 TLS 的扩展称为 Next Protocol Negotiation (NPN)。在Java 中,我们有两种不同的方式选择的基于 NPN 的协议:
* 使用 ssl_npn,NPN 的开源 SSL 提供者。
* 使用通过 Jetty 的 NPN 扩展库。

在这个例子中使用 Jetty 库。如果你想使用 ssl_npn,请参阅<https://github.com/benmmurphy/ssl_npn>项目文档

*Jetty NPN 库*

*Jetty NPN 库是一个外部的库,而不是 Netty 的本身的一部分。它用于处理 Next Protocol Negotiation, 这是用于检测客户端是否支持 SPDY。*

###  集成 Next Protocol Negotiation  

Jetty 库提供了一个接口称为 ServerProvider,确定所使用的协议和选择哪个钩子。这个的实现可能取决于不同版本的 HTTP 和 SPDY 版本的支持。下面的清单显示了将用于我们的示例应用程序的实现。

Listing 12.1 Implementation of ServerProvider

	public class DefaultServerProvider implements NextProtoNego.ServerProvider {
	    private static final List<String> PROTOCOLS =
	            Collections.unmodifiableList(Arrays.asList("spdy/2", "spdy/3", "http/1.1"));  //1
	
	    private String protocol;
	
	    @Override
	    public void unsupported() {
	        protocol = "http/1.1";   //2
	    }
	
	    @Override
	    public List<String> protocols() {
	        return PROTOCOLS;   //3
	    }
	
	    @Override
	    public void protocolSelected(String protocol) {
	        this.protocol = protocol;  //4
	    }
	
	    public String getSelectedProtocol() {
	        return protocol;  //5
	    }
	}

1. 定义所有的 ServerProvider 实现的协议
2. 设置如果 SPDY 协议失败了就转到 http/1.1 
3. 返回支持的协议的列表
4. 设置选择的协议
5. 返回选择的协议

在 ServerProvider 的实现，我们支持下面的3种协议:

* SPDY 2
* SPDY 3
* HTTP 1.1

如果客户端不支持 SPDY ，则默认使用 HTTP 1.1

#### 实现各种 ChannelHandler

第一个 ChannelInboundHandler 是用于不支持 SPDY 的情况下处理客户端 HTTP 请求，如果不支持 SPDY 就回滚使用默认的 HTTP 协议。

清单12.2显示了HTTP流量的处理程序。

Listing 12.2 Implementation that handles HTTP

	@ChannelHandler.Sharable
	public class HttpRequestHandler extends SimpleChannelInboundHandler<FullHttpRequest> {
	    @Override
	    public void channelRead0(ChannelHandlerContext ctx, FullHttpRequest request) throws Exception { //1
	        if (HttpHeaders.is100ContinueExpected(request)) {
	            send100Continue(ctx); //2
	        }
	
	        FullHttpResponse response = new DefaultFullHttpResponse(request.getProtocolVersion(), HttpResponseStatus.OK); //3
	        response.content().writeBytes(getContent().getBytes(CharsetUtil.UTF_8));  //4
	        response.headers().set(HttpHeaders.Names.CONTENT_TYPE, "text/plain; charset=UTF-8");  //5
	
	        boolean keepAlive = HttpHeaders.isKeepAlive(request);
	
	        if (keepAlive) {  //6
	            response.headers().set(HttpHeaders.Names.CONTENT_LENGTH, response.content().readableBytes());
	            response.headers().set(HttpHeaders.Names.CONNECTION, HttpHeaders.Values.KEEP_ALIVE);
	        }
	        ChannelFuture future = ctx.writeAndFlush(response);  //7
	
	        if (!keepAlive) {
	            future.addListener (ChannelFutureListener.CLOSE); //8
	        }
	    }
	
	    protected String getContent() {  //9
	        return "This content is transmitted via HTTP\r\n";
	    }
	
	    private static void send100Continue(ChannelHandlerContext ctx) {  //10
	        FullHttpResponse response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.CONTINUE);
	        ctx.writeAndFlush(response);
	    }
	
	    @Override
	    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
	            throws Exception {  //11
	        cause.printStackTrace();
	        ctx.close();
	    }
	}

1. 重写 channelRead0() ,可以被所有的接收到的 FullHttpRequest 调用
2. 检查如果接下来的响应是预期的，就写入
3. 新建 FullHttpResponse,用于对请求的响应
4. 生成响应的内容，将它写入 payload
5. 设置头文件，这样客户端就能知道如何与 响应的 payload 交互
6. 检查请求设置是否启用了 keepalive;如果是这样,将标题设置为符合HTTP RFC
7. 写响应给客户端，并获取到 Future 的引用，用于写完成时，获取到通知
8. 如果响应不是 keepalive，在写完成时关闭连接
9. 返回内容作为响应的 payload
10. Helper 方法生成了100 持续的响应，并写回给客户端
11. 若执行阶段抛出异常，则关闭管道

