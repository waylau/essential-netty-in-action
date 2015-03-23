Buffer API
====

主要包括

* ByteBuf
* ByteBufHolder

Netty 使用 reference-counting(引用计数)用来判断何时安全可以释放 ByteBuf 或 ByteBufHolder 和其他资源。这允许 Netty 使用池和其他技巧来加快速度和保持内存利用率在正常水平，你不需要做任何事情来实现这一点，但是在开发 Netty 应用程序时，你应该处理数据尽快释放池资源，特别是使用 ByteBuf 和 ByteBufHolder。

Netty 缓冲 API 提供了几个优势：

* 可以自定义缓冲类型
* 通过一个内置的复合缓冲类型实现零拷贝
* 扩展性好，比如 StringBuffer
* 不需要调用 flip() 来切换读/写模式
* 读取和写入索引分开
* 方法链
* 引用计数
* Pooling(池)