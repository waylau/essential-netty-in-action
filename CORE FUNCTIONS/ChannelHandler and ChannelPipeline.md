ChannelHandler 和 ChannelPipeline  
====

本章主要内容

- Channel
- ChannelHandler
- ChannePipeline
- ChannelHandlerContext

我们在上一章研究的 ByteBuf 是一个用来“包装”数据的容器。在本章我们将探讨这些容器是如何在应用程序中进行传输的，以及如何处理它们“包装”的数据。

Netty 在这方面提供了强大的支持。它让Channelhandler 链接在ChannelPipeline上使数据处理更加灵活和模块化。

在这一章中，下面我们会遇到各种各样 Channelhandler，ChannelPipeline 的使用案例，以及重要的相关的类Channelhandlercontext 。我们将展示如何将这些基本组成的框架可以帮助我们写干净可重用的处理实现。

