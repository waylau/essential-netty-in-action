ByteBufHolder 
====

我们经常遇到需要另外存储除有效的实际数据各种属性值。 HTTP 响应是一个很好的例子；与内容一起的字节的还有状态码, cookies,等。

Netty 提供 ByteBufHolder 处理这种常见的情况。 ByteBufHolder 还提供对于 Netty 的高级功能，如缓冲池，其中保存实际数据的 ByteBuf
可以从池中借用，如果需要还可以自动释放。

ByteBufHolder 有那么几个方法。到底层的这些支持接入数据和引用计数。表5.7所示的方法（忽略了那些从继承 ReferenceCounted 的方法）。

Table 5.7 ByteBufHolder operations

名称     | 描述
-------- | ---
data() | 返回 ByteBuf 保存的数据
copy() | 制作一个 ByteBufHolder 的拷贝，但不共享其数据(所以数据也是拷贝).

如果你想实现一个“消息对象”有效负载存储在 ByteBuf，使用ByteBufHolder 是一个好主意。