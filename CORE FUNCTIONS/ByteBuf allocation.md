ByteBuf 分配
====

本节介绍 ByteBuf 实例管理的几种方式：

###ByteBufAllocator 

为了减少分配和释放内存的开销，Netty 通过支持池类 ByteBufAllocator，可用于分配的任何 ByteBuf 我们已经描述过的类型的实例。是否使用池是由应用程序决定的，表5.8列出了 ByteBufAllocator 提供的操作。

Table 5.8 ByteBufAllocator methods

名称     | 描述
-------- | ---
buffer() buffer(int) buffer(int, int) | Return a ByteBuf with heap-based or direct data storage.
heapBuffer() heapBuffer(int) heapBuffer(int, int) | Return a ByteBuf with heap-based storage.
directBuffer() directBuffer(int) directBuffer(int, int) | Return a ByteBuf with direct storage.
compositeBuffer() compositeBuffer(int) heapCompositeBuffer() heapCompositeBuffer(int) directCompositeBuffer()directCompositeBuffer(int) | Return a CompositeByteBuf that can be expanded by adding heapbased or direct buffers.
ioBuffer() | Return a ByteBuf that will be used for I/O operations on a socket.

通过一些方法接受整型参数允许用户指定 ByteBuf 的初始和最大容量值。你可能还记得，ByteBuf 存储可以扩大到其最大容量。

得到一个 ByteBufAllocator 的引用很简单。你可以得到从 Channel （在理论上，每 Channel 可具有不同的 ByteBufAllocator ），或通过绑定到的 ChannelHandler 的 ChannelHandlerContext 得到它，用它实现了你数据处理逻辑。

下面的列表说明获得 ByteBufAllocator 的两种方式。

Listing 5.15 Obtain ByteBufAllocator reference

	Channel channel = ...;
	ByteBufAllocator allocator = channel.alloc(); //1
	....
	ChannelHandlerContext ctx = ...;
	ByteBufAllocator allocator2 = ctx.alloc(); //2
	...

1.从 channel 获得 ByteBufAllocator

2.从 ChannelHandlerContext 获得 ByteBufAllocator

Netty 提供了两种 ByteBufAllocator 的实现，一种是 PooledByteBufAllocator,用ByteBuf 实例池改进性能以及内存使用降到最低，此实现使用一个“[jemalloc](http://people.freebsd.org/~jasone/jemalloc/bsdcan2006/jemalloc.pdf)”内存分配。其他的实现不池化 ByteBuf 情况下，每次返回一个新的实例。

Netty 默认使用 PooledByteBufAllocator，我们可以通过 ChannelConfig 或通过引导设置一个不同的实现来改变。更多细节在后面讲述 ，见 [Chapter 9, "Bootstrapping Netty Applications"](Bootstrapping.md)

###Unpooled （非池化）缓存

当未引用 ByteBufAllocator 时，上面的方法无法访问到 ByteBuf。对于这个用例 Netty 提供一个实用工具类称为 Unpooled,，它提供了静态辅助方法来创建非池化的 ByteBuf 实例。表5.9列出了最重要的方法

Table 5.9 Unpooled helper class

名称     | 描述
-------- | ---
buffer() buffer(int) buffer(int, int)  | Returns an unpooled ByteBuf with heap-based storage
directBuffer() directBuffer(int) directBuffer(int, int) | Returns an unpooled ByteBuf with direct storage
wrappedBuffer()  | Returns a ByteBuf, which wraps the given data.
copiedBuffer()  | Returns a ByteBuf, which copies the given data


在 非联网项目，该 Unpooled 类也使得它更容易使用的 ByteBuf API，获得一个高性能的可扩展缓冲 API，而不需要 Netty 的其他部分的。

###ByteBufUtil 

ByteBufUtil 静态辅助方法来操作 ByteBuf，因为这个 API 是通用的，与使用池无关，这些方法已经在外面的分配类实现。

也许最有价值的是 hexDump() 方法，这个方法返回指定 ByteBuf 中可读字节的十六进制字符串，可以用于调试程序时打印 ByteBuf 的内容。一个典型的用途是记录一个 ByteBuf 的内容进行调试。十六进制字符串相比字节而言对用户更友好。 而且十六进制版本可以很容易地转换回实际字节表示。

另一个有用方法是 使用 boolean equals(ByteBuf, ByteBuf),用来比较 ByteBuf 实例是否相等。在 实现自己 ByteBuf 的子类时经常用到。

