关闭之前已经引导的客户端或服务器
====

引导您的应用程序启动并运行,但是迟早你也需要关闭它。当然你可以让 JVM处理所有退出但这不会满足“优雅”的定义，是指干净地释放资源。关闭一个Netty 的应用程序并不复杂,但有几件事要记住。

主要是记住关闭 EventLoopGroup,将处理任何悬而未决的事件和任务并随后释放所有活动线程。这只是一种叫EventLoopGroup.shutdownGracefully()。这个调用将返回一个 Future 用来通知关闭完成。注意,shutdownGracefully()也是一个异步操作,所以你需要阻塞,直到它完成或注册一个侦听器直到返回的 Future 来通知完成。

清单9.9定义了“优雅地关闭”

Listing 9.9 Graceful shutdown

	EventLoopGroup group = new NioEventLoopGroup() //1
	Bootstrap bootstrap = new Bootstrap(); //2
	bootstrap.group(group)
		.channel(NioSocketChannel.class);
	...
	...
	Future<?> future = group.shutdownGracefully(); //3
	// block until the group has shutdown
	future.sync();

1. 创建 EventLoopGroup 用于处理 I/O
2. 创建一个新的 Bootstrap 并且配置他
3. 最终优雅的关闭 EventLoopGroup 释放资源。这个也会关闭中当前使用的 Channel

或者,您可以调用 Channel.close() 显式地在所有活动管道之前调用EventLoopGroup.shutdownGracefully()。但是在所有情况下,记得关闭EventLoopGroup 本身