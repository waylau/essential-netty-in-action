构建 Netty HTTP/HTTPS 应用
====

HTTP/HTTPS 是最常见的一种协议，在智能手机里广泛应用。虽然每家公司都有一个主页,您可以通过HTTP或HTTPS访问,这不是它唯一的使用。许多组织通过 HTTP(S) 公开 WebService API ,旨在用于缓解独立的平台带来的弊端
。
让我们看一下 Netty 提供的 ChannelHandler,是如何允许您使用 HTTP 和 HTTPS 而无需编写自己的编解码器。

### HTTP Decoder, Encoder 和 Codec

HTTP 是请求-响应模式，客户端发送一个 HTTP 请求，服务就响应此请求。Netty 提供了简单的编码、解码器来简化基于这个协议的开发工作。图8.2和图8.3显示 HTTP 请求和响应的方法是如何生产和消费的

![](../images/Figure 8.2 HTTP request component parts.jpg)

1. HTTP Request 第一部分是包含的头信息
2. HttpContent 里面包含的是数据，可以后续有多个 HttpContent 部分
3. LastHttpContent 标记是 HTTP request 的结束，同时可能包含头的尾部信息
4. 完整的 HTTP request

Figure 8.2 HTTP request component parts

![](../images/Figure 8.3 HTTP response component parts.jpg)

1. HTTP response 第一部分是包含的头信息
2. HttpContent 里面包含的是数据，可以后续有多个 HttpContent 部分
3. LastHttpContent 标记是 HTTP response 的结束，同时可能包含头的尾部信息
4. 完整的 HTTP response

Figure 8.3 HTTP response component parts

如图8.2和8.3所示的 HTTP 请求/响应可能包含不止一个数据部分,它总是终止于 LastHttpContent 部分。FullHttpRequest 和FullHttpResponse 消息是特殊子类型,分别表示一个完整的请求和响应。所有类型的 HTTP 消息(FullHttpRequest ，LastHttpContent 以及那些如清单8.2所示)实现 HttpObject 接口。

表8.2概述 HTTP 解码器和编码器的处理和生产这些消息。

Table 8.2 HTTP decoder and encoder

名称 | 描述
-----|----
HttpRequestEncoder |Encodes HttpRequest , HttpContent and
LastHttpContent messages to bytes.
HttpResponseEncoder | Encodes HttpResponse, HttpContent and LastHttpContent messages to bytes.
HttpRequestDecoder | Decodes bytes into HttpRequest, HttpContent and LastHttpContent messages.
HttpResponseDecoder | Decodes bytes into HttpResponse, HttpContent and LastHttpContent messages.


清单8.2所示的是将支持 HTTP 添加到您的应用程序是多么简单。仅仅添加正确的 ChannelHandler 到 ChannelPipeline 中

Listing 8.2 Add support for HTTP

	public class HttpPipelineInitializer extends ChannelInitializer<Channel> {
	
	    private final boolean client;
	
	    public HttpPipelineInitializer(boolean client) {
	        this.client = client;
	    }
	
	    @Override
	    protected void initChannel(Channel ch) throws Exception {
	        ChannelPipeline pipeline = ch.pipeline();
	        if (client) {
	            pipeline.addLast("decoder", new HttpResponseDecoder());  //1
	            pipeline.addLast("encoder", new HttpRequestEncoder());  //2
	        } else {
	            pipeline.addLast("decoder", new HttpRequestDecoder());  //3
	            pipeline.addLast("encoder", new HttpResponseEncoder());  //4
	        }
	    }
	}

1. client: 添加 HttpResponseDecoder 用于处理来自 server 响应
2. client: 添加 HttpRequestEncoder 用于发送请求到 server
3. server: 添加 HttpRequestDecoder 用于接收来自 client 的请求
4. server: 添加 HttpResponseEncoder 用来发送响应给 client
