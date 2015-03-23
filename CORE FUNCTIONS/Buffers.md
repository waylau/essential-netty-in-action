Buffer（缓冲）
=====

正如我们先前所指出的，网络数据的基本单位永远是 byte(字节)。Java NIO 提供 ByteBuffer 作为字节的容器，但这个类是过于复杂，有点
难以使用。

Netty 中 ByteBuffer 替代是 ByteBuf，一个强大的实现，解决
JDK 的 API 的限制，以及为网络应用程序开发者一个更好的工具。
但 ByteBuf 并不仅仅暴露操作一个字节序列的方法;这也是专门的Netty 的 ChannelPipeline 的语义设计。

在本章中，我们会说明相比于 JDK 的 API，ByteBuf 所提供的卓越的功能和灵活性。这也将使我们能够更好地理解了 Netty 的数据处理。