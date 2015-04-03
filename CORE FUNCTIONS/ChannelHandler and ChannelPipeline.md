ChannelHandler 和 ChannelPipeline  
====

本章主要内容

- Channel
- ChannelHandler
- ChannePipeline
- ChannelHandlerContext

我们在上一章研究的 bytebuf 是一个容器用来“包装”数据。在本章我们将探讨这些容器如何通过应用程序来移动，传入和传出，以及他们的内容是如何处理的。

Netty 提供了应用开发的数据处理方面的强大支持。我们已经看到了channelhandler 如何链接在一起 ChannelPipeline 使用结构处理更加灵活和模块化。

在这一章中，下面我们会遇到各种各样 Channelhandler，ChannelPipeline 的使用案例，以及重要的相关的类Channelhandlercontext 。我们将展示如何将这些基本组成的框架可以帮助我们写干净可重用的处理实现。

