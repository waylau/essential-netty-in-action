解码分隔符和基于长度的协议
====

使用 Netty 时会遇到需要解码以分隔符和长度为基础的协议，本节讲解Netty 如何解码这些协议。

### 分隔符协议
        
经常需要处理分隔符协议或创建基于它们的协议，例如[SMTP](http://www.ietf.org/rfc/rfc2821.txt)、[POP3](http://www.ietf.org/rfc/rfc1939.txt)、[IMAP](http://tools.ietf.org/html/rfc3501)、[Telnet](http://tools.ietf.org/search/rfc854)等等。Netty 附带的解码器可以很容易的提取一些序列分隔：

Table 8.5 Decoders for handling delimited and length-based protocols

名称 | 描述
-----|----
DelimiterBasedFrameDecoder | 接收ByteBuf由一个或多个分隔符拆分，如NUL或换行符
LineBasedFrameDecoder| 接收ByteBuf以分割线结束，如"\n"和"\r\n"

下图显示了使用"\r\n"分隔符的处理：

![](../images/Figure 8.5 Handling delimited frames.jpg)

1. 字节流
2. 第一帧
3. 第二帧

Figure 8.5 Handling delimited frames

下面展示了如何用 LineBasedFrameDecoder 处理

Listing 8.8 Handling line-delimited frames


