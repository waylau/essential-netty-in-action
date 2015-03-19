近距离观察 ChannelHandler
=======

正如我们之前所说，有很多不同类型的 ChannelHandler 。每个 ChannelHandler 做什么取决于其超类。 Netty 提供了一些默认的处理程序实现形式的“adapter（适配器）”类。这些旨在简化开发处理逻辑。我们已经看到，在 pipeline 中每个的 ChannelHandler 负责转发事件到链中的下一个处理器。这些适配器类（及其子类）会自动帮你实现，所以你只需要实现该特定的方法和事件。

*为什么用适配器？*

*有几个适配器类，可以减少编写自定义 ChannelHandlers ，因为他们提供对应接口的所有方法的默认实现。（也有类似的适配器，用于创建编码器和解码器，这我们将在稍后讨论。）这些都是创建自定义处理器时，会经常调用的适配器：ChannelHandlerAdapter、ChannelInboundHandlerAdapter、ChannelOutboundHandlerAdapter、ChannelDuplexHandlerAdapter*

下面解释下三个 ChannelHandler 的子类型：编码器、解码器以及 ChannelInboundHandlerAdapter 的子类SimpleChannelInboundHandler<T>

###编码器、解码器

当您发送或接收消息时，Netty 数据转换就发生了。入站消息将从字节转为一个Java对象;也就是说，“解码”。如果该消息是出站相反会发生：“编码”，从一个Java对象转为字节。其原因是简单的：网络数据是一系列字节，因此需要从那类型进行转换。

不同类型的抽象类用于提供编码器和解码器的，这取决于手头的任务。例如，应用程序可能并不需要马上将消息转为字节。相反，该​​消息将被转换
一些其他格式。一个编码器将仍然可以使用，但它也将衍生自不同的超类，

在一般情况下，基类将有一个名字类似 ByteToMessageDecoder 或
MessageToByteEncoder。在一种特殊类型的情况下，你可能会发现类似
ProtobufEncoder 和 ProtobufDecoder，用于支持谷歌的 protocol buffer。

严格地说，其他处理器可以做编码器和解码器能做的事。但正如适配器类简化创建通道处理器，所有的编码器/解码器适配器类 都实现自 ChannelInboundHandler 或 ChannelOutboundHandler。

对于入站数据，channelRead 方法/事件被覆盖。这种方法在每个消息从入站 Channel 读入时调用。该方法将调用特定解码器的“解码”方法，并将解码后的消息转发到管道中下个的 ChannelInboundHandler。

出站消息是类似的。编码器将消息转为字节，转发到下个的 ChannelOutboundHandler。

###SimpleChannelHandler 

也许最常见的处理器是接收到解码后的消息并应用一些业务逻辑到这些数据。要创建这样一个 ChannelHandler，你只需要扩展基类SimpleChannelInboundHandler<T> 其中 T 是想要进行处理的类型。这样的处理器，你将覆盖基类的一个或多个方法，将获得被作为输入参数传递所有方法的 ChannelHandlerContext 的引用。

在这种类型的处理器方法中的最重要是 channelRead0(ChannelHandlerContext，T)。在这个调用中，T 是将要处理的消息。
你怎么做，完全取决于你，但无论如何你不能阻塞 I/O线程，因为这可能是不利于高性能。

*阻塞操作*

*I/O 线程一定不能完全阻塞，因此禁止任何直接阻塞操作在你的 ChannelHandler， 有一种方法来实现这一要求。你可以指定一个 EventExecutorGroup 当添加 ChannelHandler 到ChannelPipeline。此 EventExecutorGroup  将用于获得EventExecutor，将执行所有的 ChannelHandler 的方法。这EventExecutor 将从 I/O 线程使用不同的线程，从而释放EventLoop。*