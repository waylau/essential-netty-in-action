WebSocket
====

本章涵盖了如下内容：

* WebSockets
* ChannelHandler, Decoder 和 Encoder
* 引导你的应用程序

[real-time web](http://en.wikipedia.org/wiki/Real-time_web)（实时网络）是一组技术和实践，使用户能够及时接收
到作者发表的信息，而不需要用户用他们的软件定期检查更新源。

HTTP 的请求/响应的设计并不能满足实时的需求，而 WebSocket 协议从设计以来就提供双向数据传输，允许客户和服务器在任何时间发送消息，并要求它们能够异步处理消息。最新的浏览器都支持 WebSocket 的作为
HTML5 的客户端 API。

Netty 中的 [WebSocket](http://tools.ietf.org/html/rfc6455) 支持包括正在使用的所有主要的实现，所以采用它在你的下一个应用程序中非常简单。像往常一样使用完全的协议，而不必担心其内部实现细节。
我们将通过开发基于 WebSocket 的实时聊天应用证明这一点。

