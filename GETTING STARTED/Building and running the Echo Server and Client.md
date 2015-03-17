编译和运行 Echo 服务器和客户端
=========

###编译

*本例涉及到多模块 Maven 项目的组织*

在例子 chapter2 目录下，执行

	mvn clean package

输出如下

Listing 2.6 Build Output

	chapter2>mvn clean package
	[INFO] Scanning for projects...
	[INFO] --------------------------------------------------------------------
	[INFO] Reactor Build Order:
	[INFO]
	[INFO] Echo Client and Server
	[INFO] Echo Client
	[INFO] Echo Server
	[INFO]
	[INFO] --------------------------------------------------------------------
	[INFO] Building Echo Client and Server 1.0-SNAPSHOT
	[INFO] --------------------------------------------------------------------
	[INFO]
	[INFO] --- maven-clean-plugin:2.5:clean (default-clean) @ echo-parent ---
	[INFO]
	[INFO] --------------------------------------------------------------------
	[INFO] Building Echo Client 1.0-SNAPSHOT
	[INFO] --------------------------------------------------------------------
	[INFO]
	[INFO] --- maven-clean-plugin:2.5:clean (default-clean) @ echo-client ---
	[INFO]
	[INFO] --- maven-compiler-plugin:3.1:compile (default-compile)
	@ echo-client ---
	[INFO] Changes detected - recompiling the module!
	[INFO] --------------------------------------------------------------------
	[INFO] Reactor Summary:
	[INFO]
	[INFO] Echo Client and Server ......................... SUCCESS [ 0.118 s]
	[INFO] Echo Client .................................... SUCCESS [ 1.219 s]
	[INFO] Echo Server .................................... SUCCESS [ 0.110 s]
	[INFO] --------------------------------------------------------------------
	[INFO] BUILD SUCCESS
	[INFO] --------------------------------------------------------------------
	[INFO] Total time: 1.561 s
	[INFO] Finished at: 2014-06-08T17:39:15-05:00
	[INFO] Final Memory: 14M/245M
	[INFO] --------------------------------------------------------------------

注意事项：

* Maven Reactor 构建顺序：先是 父 POM，然后是子项目
* Netty artifact 没在用户的本地存储库中找到，所以 Maven 就会从互联网上下载
* clean 和 compile 在构建生命周期的运行。事后 mavensurefire-plugin
插件运行，但不会有测试类存在。最后 mavenjar-plugin 执行

这段说明了项目已经成功编译。

###运行 Echo 服务器 和 客户端

我们使用 exec-maven-plugin 来运行项目。

在 chapter2/Server 目录，执行

	mvn exec:java

输出如下：

	[INFO] Scanning for projects...
	[INFO]
	[INFO] --------------------------------------------------------------------
	[INFO] Building Echo Server 1.0-SNAPSHOT
	[INFO] --------------------------------------------------------------------
	[INFO]
	[INFO] >>> exec-maven-plugin:1.2.1:java (default-cli) @ echo-server >>>
	[INFO]
	[INFO] <<< exec-maven-plugin:1.2.1:java (default-cli) @ echo-server <<<
	[INFO]
	[INFO] --- exec-maven-plugin:1.2.1:java (default-cli) @ echo-server ---
	nettyinaction.echo.EchoServer started and listening for connections on
	/0:0:0:0:0:0:0:0:9999

在 chapter2/Client 目录，执行

	mvn exec:java

输出如下：

	[INFO] Scanning for projects...
	[INFO]
	[INFO] --------------------------------------------------------------------
	[INFO] Building Echo Client 1.0-SNAPSHOT
	[INFO] --------------------------------------------------------------------
	[INFO]
	[INFO] >>> exec-maven-plugin:1.2.1:java (default-cli) @ echo-client >>>
	[INFO]
	[INFO] <<< exec-maven-plugin:1.2.1:java (default-cli) @ echo-client <<<
	[INFO]
	[INFO] --- exec-maven-plugin:1.2.1:java (default-cli) @ echo-client ---
	Client received: Netty rocks!
	[INFO] --------------------------------------------------------------------
	[INFO] BUILD SUCCESS
	[INFO] --------------------------------------------------------------------
	[INFO] Total time: 3.907 s
	[INFO] Finished at: 2014-06-08T18:26:14-05:00
	[INFO] Final Memory: 8M/245M
	[INFO] --------------------------------------------------------------------

在服务器的控制台输出：

	Server received: Netty rocks!

发生了什么事：

* 客户端连接后，它发送消息：“Netty rocks！”
* 服务器输出接收到消息并将其返回给客户端
* 客户输出接收到的消息并退出。

每次运行客户端，你会看到在服务器的控制台输出：

	Server received: Netty rocks!

现在，我们看下错误的情况。在控制台 输入 Ctrl-C 来关闭服务器。而后运行客户端，此时输出如下：

	[INFO] Scanning for projects...
	[INFO]
	[INFO] --------------------------------------------------------------------
	[INFO] Building Echo Client 1.0-SNAPSHOT
	[INFO] --------------------------------------------------------------------
	[INFO]
	[INFO] >>> exec-maven-plugin:1.2.1:java (default-cli) @ echo-client >>>
	[INFO]
	[INFO] <<< exec-maven-plugin:1.2.1:java (default-cli) @ echo-client <<<
	[INFO]
	[INFO] --- exec-maven-plugin:1.2.1:java (default-cli) @ echo-client ---
	[WARNING]
	java.lang.reflect.InvocationTargetException
		at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
		at sun.reflect.NativeMethodAccessorImpl.invoke
		(NativeMethodAccessorImpl.java:57)
		at sun.reflect.DelegatingMethodAccessorImpl.invoke
		(DelegatingMethodAccessorImpl.java:43)
		at java.lang.reflect.Method.invoke(Method.java:606)
		at org.codehaus.mojo.exec.ExecJavaMojo$1.run(ExecJavaMojo.java:297)
		at java.lang.Thread.run(Thread.java:744)
		Caused by: java.net.ConnectException: Connection refused:
		no further information: localhost/127.0.0.1:9999
		at sun.nio.ch.SocketChannelImpl.checkConnect(Native Method)
		at sun.nio.ch.SocketChannelImpl.finishConnect
		(SocketChannelImpl.java:739)
		at io.netty.channel.socket.nio.NioSocketChannel
		.doFinishConnect(NioSocketChannel.java:191)
		at io.netty.channel.nio.
		AbstractNioChannel$AbstractNioUnsafe.finishConnect(
		AbstractNioChannel.java:279)
		at io.netty.channel.nio.NioEventLoop
		.processSelectedKey(NioEventLoop.java:511)
		at io.netty.channel.nio.NioEventLoop
		.processSelectedKeysOptimized(NioEventLoop.java:461)
		at io.netty.channel.nio.NioEventLoop
		.processSelectedKeys(NioEventLoop.java:378)
		at io.netty.channel.nio.NioEventLoop.run(NioEventLoop.java:350)
		at io.netty.util.concurrent
		.SingleThreadEventExecutor$2.run
		(SingleThreadEventExecutor.java:101)
		... 1 more
	[INFO] --------------------------------------------------------------------
	[INFO] BUILD FAILURE
	[INFO] --------------------------------------------------------------------
	[INFO] Total time: 3.728 s
	[INFO] Finished at: 2014-06-08T18:49:13-05:00
	[INFO] Final Memory: 8M/245M
	[INFO] --------------------------------------------------------------------
	[ERROR] Failed to execute goal org.codehaus.mojo:exec-maven-plugin:1.2.1:java
		(default-cli) on project echo-client: An exception occured while executing the
		Java class. null: InvocationTargetException: Connection refused: no further
		information:
		localhost/127.0.0.1:9999 -> [Help 1]
		 
发生了啥？客户端尝试连接服务器，但服务器是关闭的，所以引发了一个 java.net.ConnectException ，这个异常被 EchoClientHandler 的 exceptionCaught() 触发，打印出异常信息，并关闭 channel

