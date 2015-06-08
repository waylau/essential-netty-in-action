WebSocket
====

本章涵盖了如下内容：

* WebSockets
* ChannelHandler, Decoder 和 Encoder
* 引导你的应用程序

[real-time web](http://en.wikipedia.org/wiki/Real-time_web)（实时web）是一组技术和实践，使用户能够实时地接收
到作者发布的信息，而不需要用户用他们的软件定期检查更新源。

HTTP 的请求/响应的设计并不能满足实时的需求，而 WebSocket 协议从设计以来就提供双向数据传输，允许客户和服务器在任何时间发送消息，并要求它们能够异步处理消息。最新的浏览器都将 WebSockets 作为HTML5的一种客户端API来支持的。

Netty 中对于 [WebSocket](http://tools.ietf.org/html/rfc6455) 的支持包括正在使用的所有主要的实现，所以在你的下一个应用程序中采用它会非常简单。像往常使用Netty一样，你可以充分利用这种协议，而不必担心其内部实现细节。
我们将通过开发基于 WebSocket 的实时聊天应用证明这一点。

