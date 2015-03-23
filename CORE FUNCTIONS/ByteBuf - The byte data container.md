ByteBuf - 字节数据的容器
====

网络进行交互时，需要以字节码发送/接收数据。由于各种原因，一个高效、方便、易用的数据接口是必须的，而 Netty 的 ByteBuf 满足这些需求。

ByteBuf 是一个很好的经过优化的数据容器，我们可以将字节数据有效的添加到 ByteBuf 中或从 ByteBuf 中获取数据。ByteBuf 有2部分：一个用于读，一个用于写。我们可以按顺序的读取数据，并且可以跳到开始重新读一遍。所有的数据操作，我们只需要做的是调整读取数据索引和再次开始通过 get() 方法进行读操作。

###ByteBuf 如何在工作？

写入数据到 ByteBuf 后，writerIndex（写入索引）增加。开始读字节后，readerIndex（读取索引）增加。你可以读取字节，直到写入索引和读取索引处理相同的位置，ByteBuf 变为不可读。当访问数据超过数组的最后位，则会抛出 IndexOutOfBoundsException。

调用 ByteBuf 的 "read" 或 "write" 开头的任何方法都会提升 相应的索引。另一方面，"set" 、 "get"操作字节将不会移动指数；他们只操作相关的通过参数传入方法的索引。

ByteBuf 的默认最大容量限制是 Integer.MAX_VALUE，写入时若超出这个值将会导致一个异常。

ByteBuf 类似于一个字节数组，最大的区别是读和写的索引可以用来控制对缓冲区数据的访问。下图显示了一个容量为16的空的 ByteBuf  的布局和状态，writerIndex 和 readerIndex 都在索引位置 0 ：

Figure 5.1 A 16-byte ByteBuf with its indices set to 0

![](../images/Figure 5.1 A 16-byte ByteBuf with its indices set to 0.jpg)

###ByteBuf 使用模式

####HEAP BUFFER(堆缓冲区)

最常用的类型是 ByteBuf 将数据存储在 JVM 的堆空间，这是通过将数据存储在数组的实现。堆缓冲区可以快速分配，当不使用时也可以快速释放。它还提供了直接访问数组的方法，通过 ByteBuf.array() 来获取 byte[]数据。这种方法，清单5.1中示出，是非常适合的情况下
你必须处理遗留数据。

Listing 5.1 Backing array

	ByteBuf heapBuf = ...;
    if (heapBuf.hasArray()) {				//1
        byte[] array = heapBuf.array();		//2
        int offset = heapBuf.arrayOffset() + heapBuf.readerIndex();				//3
        int length = heapBuf.readableBytes();//4
        handleArray(array, offset, length); //5
    }


1.检查 ByteBuf 是否有支持数组。

2.如果这样得到的引用数组。

3.计算第一字节的偏移量。

4.获取可读的字节数。

5.使用数组，偏移量和长度作为调用方法的参数。

注意：

* 访问非堆缓冲区 ByteBuf 的数组会导致UnsupportedOperationException， 可以使用 ByteBuf.hasArray()来检查是否支持访问数组。
* 这个用法与 JDK 的 ByteBuffer 类似

####DIRECT BUFFER(直接缓冲区)

“直接缓冲区”是另一个 ByteBuf 模式。对象的所有内存分配发生在
堆，对不对？好吧，并非总是如此。在 JDK1.4 中被引入 NIO 的ByteBuffer 类允许一个 JVM 实现通过本地调用分配内存，其目的是
为了避免之前复制缓冲区的内容（或）中间缓冲区（或
后）的底层操作系统的本机我的一个在每次调用/ O
操作。
。 。 。
直接缓冲区的内容可以驻留的正常垃圾回收以外
堆。
这就解释了为什么“直接缓冲区”是理想的数据传输通过套接字。如果你的数据是
包含在堆中分配的缓冲区，JVM实际上将复制您的缓冲区，直接缓冲区
之前在内部通过套接字发送。


直接缓冲区，在堆之外直接分配内存。直接缓冲区不会占用堆空间容量，使用时应该考虑到应用程序要使用的最大内存容量以及如何限制它。直接缓冲区在使用Socket传递数据时性能很好，因为若使用间接缓冲区，JVM会先将数据复制到直接缓冲区再进行传递；但是直接缓冲区的缺点是在分配内存空间和释放内存时比堆缓冲区更复杂，而Netty使用内存池来解决这样的问题，这也是Netty使用内存池的原因之一。直接缓冲区不支持数组访问数据，但是我们可以间接的访问数据数组，如下面代码：