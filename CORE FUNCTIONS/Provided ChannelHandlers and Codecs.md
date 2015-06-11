已经提供的 ChannelHandler 和 Codec
====

本章介绍

* 使用 SSL/TLS 加密 Netty 程序
* 构建 Netty HTTP/HTTPS 程序
* 处理空闲连接和超时
* 解码分隔符和基于长度的协议
* 写大数据
* 序列化数据

Netty 提供了很多共同协议的编解码器和处理程序,您可以几乎“开箱即用”的使用他们,而无需花在相当乏味的基础设施问题。在这一章里,我们将探索这些工具和他们的好处。这包括支持 SSL/TLS,WebSocket 和 谷歌SPDY,通过数据压缩使 HTTP 有更好的性能。