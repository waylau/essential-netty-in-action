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

	public class HttpRequestHandler extends SimpleChannelInboundHandler<FullHttpRequest> {	//1
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
	            ctx.fireChannelRead(request.retain());					//2
	        } else {
	            if (HttpHeaders.is100ContinueExpected(request)) {
	                send100Continue(ctx);								//3
	            }
	
	            RandomAccessFile file = new RandomAccessFile(INDEX, "r");//4
	
	            HttpResponse response = new DefaultHttpResponse(request.getProtocolVersion(), HttpResponseStatus.OK);
	            response.headers().set(HttpHeaders.Names.CONTENT_TYPE, "text/html; charset=UTF-8");
	
	            boolean keepAlive = HttpHeaders.isKeepAlive(request);
	
	            if (keepAlive) {										//5
	                response.headers().set(HttpHeaders.Names.CONTENT_LENGTH, file.length());
	                response.headers().set(HttpHeaders.Names.CONNECTION, HttpHeaders.Values.KEEP_ALIVE);
	            }
	            ctx.write(response);					//6
	
	            if (ctx.pipeline().get(SslHandler.class) == null) {		//7
	                ctx.write(new DefaultFileRegion(file.getChannel(), 0, file.length()));
	            } else {
	                ctx.write(new ChunkedNioFile(file.getChannel()));
	            }
	            ChannelFuture future = ctx.writeAndFlush(LastHttpContent.EMPTY_LAST_CONTENT);			//8
	            if (!keepAlive) {
	                future.addListener(ChannelFutureListener.CLOSE);		//9
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

1.扩展 SimpleChannelInboundHandler 用于处理 FullHttpRequest信息

2.如果请求是 WebSocket 升级，递增引用计数器（保留）并且将它传递给在  ChannelPipeline 中的下个 ChannelInboundHandler 

3.处理符合 HTTP 1.1的 "100 Continue" 请求
 
4.读取 index.html

5.判断 keepalive 是否在请求头里面

6.写 HttpResponse 到客户端

7.写 index.html 到客户端，判断 SslHandler 是否在 ChannelPipeline 来决定是使用 DefaultFileRegion 还是 ChunkedNioFile

8.写并刷新 LastHttpContent 到客户端，标记响应完成

9.如果 keepalive 没有要求，当写完成时，关闭 Channel

HttpRequestHandler 做了下面几件事，

* 如果该 HTTP 请求被发送到URI “/ws”，调用 FullHttpRequest 上的 retain()，并通过调用 fireChannelRead(msg) 转发到下一个 ChannelInboundHandler。retain() 是必要的，因为 channelRead() 完成后，它会调用 FullHttpRequest 上的 release() 来释放其资源。 （请参考我们先前的 SimpleChannelInboundHandler 在第6章中讨论）
* 如果客户端发送的 HTTP 1.1 头是“Expect: 100-continue” ，将发送“100 Continue”的响应。
* 在 头被设置后，写一个 HttpResponse 返回给客户端。注意，这是不是 FullHttpResponse，唯一的反应的第一部分。此外，我们不使用 writeAndFlush() 在这里 - 这个是在最后完成。
* 如果没有加密也不压缩，要达到最大的效率可以是通过存储  index.html 的内容在一个 DefaultFileRegion 实现。这将利用零拷贝来执行传输。出于这个原因，我们检查，看看是否有一个 SslHandler 在 ChannelPipeline 中。另外，我们使用 ChunkedNioFile。
* 写 LastHttpContent 来标记响应的结束，并终止它
* 如果不要求 keepalive ，添加 ChannelFutureListener 到 ChannelFuture 对象的最后写入，并关闭连接。注意，这里我们调用 writeAndFlush() 来刷新所有以前写的信息。

这里展示了应用程序的第一部分，用来处理纯的 HTTP 请求和响应。接下来我们将处理 WebSocket 的 frame（帧），用来发送聊天消息。

*WebSocket frame*

*WebSockets 在“帧”里面来发送数据，其中每一个都代表了一个消息的一部分。一个完整的消息可以利用了多个帧。*

###处理 WebSocket frame

WebSocket "Request for Comments" (RFC) 定义了六中不同的 frame; Netty 给他们每个都提供了一个 POJO 实现 ，见下表：

Table 11.1 WebSocketFrame types

名称 | 描述
----- | ----
BinaryWebSocketFrame  |  contains binary data
TextWebSocketFrame  | contains text data
ContinuationWebSocketFrame  | contains text or binary data that belongs to a previous BinaryWebSocketFrame or TextWebSocketFrame
CloseWebSocketFrame  | represents a CLOSE request and contains close status code and a phrase
PingWebSocketFrame  | requests the transmission of a PongWebSocketFrame
PongWebSocketFrame  | sent as a response to a PingWebSocketFrame


我们的程序只需要使用下面4个框架：
* CloseWebSocketFrame
* PingWebSocketFrame
* PongWebSocketFrame
* TextWebSocketFrame

我们只需要显示处理 TextWebSocketFrame，其他的会自动由 WebSocketServerProtocolHandler 处理。

下面代码展示了 ChannelInboundHandler 处理 TextWebSocketFrame，同时也将跟踪在 ChannelGroup 中所有活动的 WebSocket 连接

Listing 11.2 Handles Text frames



