添加 WebSocket 支持
====

WebSocket 通过“[Upgrade handshake](https://developer.mozilla.org/en-US/docs/HTTP/Protocol_upgrade_mechanism)（升级握手）”从标准的 HTTP 或HTTPS 协议转为 WebSocket。因此，使用 WebSocket 的应用程序将始终
以 HTTP/S 开始，然后进行升级。在什么时候发生这种情况是特定于
申请;它可以是在启动时，或当一个特定的URL已被请求。

在我们的应用中，当 URL 请求以“/ws”结束时，我们才升级协议为WebSocket。否则，服务器将使用基本的 HTTP/S。一旦升级连接将使用的WebSocket 传输的所有数据。

下面看下服务器的逻辑

Figure 11.2 Server logic

![](../images/Figure 11.2 Server logic.jpg)

＃1客户端/用户连接到服务器并加入聊天

＃2 HTTP 请求页面或 WebSocket 升级握手

＃3服务器处理所有客户端/用户

＃4响应 URI “/”的请求，转到 index.html

＃5如果访问的是 URI“/ws” ，处理 WebSocket 升级握手

＃6升级握手完成后 ，通过 WebSocket 发送聊天消息

###处理 HTTP 请求

本节实现处理 HTTP 请求，生成页面用来访问“聊天室”，并且显示发送的消息。

下面代码 HttpRequestHandler,是一个用来处理 FullHttpRequest 消息的 ChannelInboundHandler 的实现。注意，忽略 "/ws"
URI 的请求。

Listing 11.1 HTTPRequestHandler

	public class HttpRequestHandler extends SimpleChannelInboundHandler<FullHttpRequest> {
	    private final String wsUri;
	    private static final File INDEX;
	
	    static {
	        URL location = HttpRequestHandler.class.getProtectionDomain().getCodeSource().getLocation();
	        try {
	            String path = location.toURI() + "index.html";
	            path = !path.contains("file:") ? path : path.substring(5);
	            INDEX = new File(path);
	        } catch (URISyntaxException e) {
	            throw new IllegalStateException("Unable to locate index.html", e);
	        }
	    }
	
	    public HttpRequestHandler(String wsUri) {
	        this.wsUri = wsUri;
	    }
	
	    @Override
	    public void channelRead0(ChannelHandlerContext ctx, FullHttpRequest request) throws Exception {
	        if (wsUri.equalsIgnoreCase(request.getUri())) {
	            ctx.fireChannelRead(request.retain());
	        } else {
	            if (HttpHeaders.is100ContinueExpected(request)) {
	                send100Continue(ctx);
	            }
	
	            RandomAccessFile file = new RandomAccessFile(INDEX, "r");
	
	            HttpResponse response = new DefaultHttpResponse(request.getProtocolVersion(), HttpResponseStatus.OK);
	            response.headers().set(HttpHeaders.Names.CONTENT_TYPE, "text/html; charset=UTF-8");
	
	            boolean keepAlive = HttpHeaders.isKeepAlive(request);
	
	            if (keepAlive) {
	                response.headers().set(HttpHeaders.Names.CONTENT_LENGTH, file.length());
	                response.headers().set(HttpHeaders.Names.CONNECTION, HttpHeaders.Values.KEEP_ALIVE);
	            }
	            ctx.write(response);
	
	            if (ctx.pipeline().get(SslHandler.class) == null) {
	                ctx.write(new DefaultFileRegion(file.getChannel(), 0, file.length()));
	            } else {
	                ctx.write(new ChunkedNioFile(file.getChannel()));
	            }
	            ChannelFuture future = ctx.writeAndFlush(LastHttpContent.EMPTY_LAST_CONTENT);
	            if (!keepAlive) {
	                future.addListener(ChannelFutureListener.CLOSE);
	            }
	        }
	    }
	
	    private static void send100Continue(ChannelHandlerContext ctx) {
	        FullHttpResponse response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.CONTINUE);
	        ctx.writeAndFlush(response);
	    }
	
	    @Override
	    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
	            throws Exception {
	        cause.printStackTrace();
	        ctx.close();
	    }
	}
