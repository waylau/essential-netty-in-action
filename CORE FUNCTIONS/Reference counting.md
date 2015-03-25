引用计数器
====

Netty 4 引入了 引用计数器给 ByteBuf 和 ByteBufHolder（两者都实现了 ReferenceCounted 接口）

引用计数本身并不复杂;它在特定的对象上跟踪引用的数目。实现了ReferenceCounted 的类的实例会通常开始于一个活动的引用计数器为 1。活动的引用计数器大于0的对象被保证不被释放。当数量引用减少到0，该实例将被释放。需要注意的是“释放”的语义是特定于具体的实现。最起码，一个对象，它已被释放应不再可用。

这种技术就是诸如 PooledByteBufAllocator 这种减少内存分配开销的池化的精髓部分。

Listing 5.16 Reference counting

	Channel channel = ...;
	ByteBufAllocator allocator = channel.alloc(); //1
	....
	ByteBuf buffer = allocator.directBuffer(); //2
	assert buffer.refCnt() == 1; //3
	...

1.从 channel 获取 ByteBufAllocator

2.从 ByteBufAllocator 分配一个 ByteBuf

3.检查引用计数器是否是 1

Listing 5.17 Release reference counted object

	ByteBuf buffer = ...;
	boolean released = buffer.release(); //1
	...

1.release（）将会递减对象引用的数目。当这个引用计数达到0时，对象已被释放，并且该方法返回 true。

如果尝试访问已经释放的对象，将会抛出 IllegalReferenceCountException 异常。

需要注意的是一个特定的类可以定义自己独特的方式其释放计数的“规则”。
例如，release() 可以将引用计数器直接计为 0 而不管当前引用的对象数目。

*谁负责 release？*

*在一般情况下，最后访问的对象负责释放它。在第6章我们会解释 ChannelHandler 和 ChannelPipeline 的相关概念。*

