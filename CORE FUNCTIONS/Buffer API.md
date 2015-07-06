Buffer API
====

主要包括

* ByteBuf
* ByteBufHolder

Netty 使用 reference-counting(引用计数)来判断何时可以释放 ByteBuf 或 ByteBufHolder 和其他相关资源，从而可以利用池和其他技巧来提高性能和降低内存的消耗。这一点上不需要开发人员做任何事情，但是在开发 Netty 应用程序时，尤其是使用 ByteBuf 和 ByteBufHolder 时，你应该尽可能早地释放池资源。
Netty 缓冲 API 提供了几个优势：

* 可以自定义缓冲类型
* 通过一个内置的复合缓冲类型实现零拷贝
* 扩展性好，比如 StringBuilder
* 不需要调用 flip() 来切换读/写模式
* 读取和写入索引分开
* 方法链
* 引用计数
* Pooling(池)