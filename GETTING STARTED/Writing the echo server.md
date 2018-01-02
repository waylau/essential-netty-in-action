写一个 echo 服务器
======

Netty 实现的 echo 服务器都需要下面这些：

* 一个服务器 handler：这个组件实现了服务器的业务逻辑，决定了连接创建后和接收到信息后该如何处理
* Bootstrapping： 这个是配置服务器的启动代码。最少需要设置服务器绑定的端口，用来监听连接请求。

###通过 ChannelHandler 来实现服务器的逻辑

Echo Server 将会将接受到的数据的拷贝发送给客户端。因此，我们需要实现 ChannelInboundHandler 接口，用来定义处理入站事件的方法。由于我们的应用很简单，只需要继承 ChannelInboundHandlerAdapter 就行了。这个类 提供了默认 ChannelInboundHandler 的实现，所以只需要覆盖下面的方法：


* channelRead() - 每个信息入站都会调用
* channelReadComplete() - 通知处理器最后的 channelRead() 是当前批处理中的最后一条消息时调用
* exceptionCaught()- 读操作时捕获到异常时调用

EchoServerHandler 代码如下：

Listing 2.2 EchoServerHandler
	
	@Sharable										//1
	public class EchoServerHandler extends
	        ChannelInboundHandlerAdapter {
	
	    @Override
	    public void channelRead(ChannelHandlerContext ctx,
	        Object msg) {
	        ByteBuf in = (ByteBuf) msg;
	        System.out.println("Server received: " + in.toString(CharsetUtil.UTF_8));		//2
	        ctx.write(in);							//3
	    }
	
	    @Override
	    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
	        ctx.writeAndFlush(Unpooled.EMPTY_BUFFER)//4
			.addListener(ChannelFutureListener.CLOSE);
	    }
	
	    @Override
	    public void exceptionCaught(ChannelHandlerContext ctx,
	        Throwable cause) {
	        cause.printStackTrace();				//5
	        ctx.close();							//6
	    }
	}

1.`@Sharable` 标识这类的实例之间可以在 channel 里面共享

2.日志消息输出到控制台

3.将所接收的消息返回给发送者。注意，这还没有冲刷数据

4.冲刷所有待审消息到远程节点。关闭通道后，操作完成

5.打印异常堆栈跟踪

6.关闭通道

这种使用 ChannelHandler 的方式体现了关注点分离的设计原则，并简化业务逻辑的迭代开发的要求。处理程序很简单;它的每一个方法可以覆盖到“hook（钩子）”在活动周期适当的点。很显然，我们覆盖 channelRead因为我们需要处理所有接收到的数据。

覆盖 exceptionCaught 使我们能够应对任何 Throwable 的子类型。在这种情况下我们记录，并关闭所有可能处于未知状态的连接。它通常是难以
从连接错误中恢复，所以干脆关闭远程连接。当然，也有可能的情况是可以从错误中恢复的，所以可以用一个更复杂的措施来尝试识别和处理
这样的情况。

*如果异常没有被捕获，会发生什么？*

*每个 Channel 都有一个关联的 ChannelPipeline，它代表了 ChannelHandler 实例的链。适配器处理的实现只是将一个处理方法调用转发到链中的下一个处理器。因此，如果一个 Netty 应用程序不覆盖exceptionCaught ，那么这些错误将最终到达 ChannelPipeline，并且结束警告将被记录。出于这个原因，你应该提供至少一个 实现 exceptionCaught 的 ChannelHandler。*

关键点要牢记：

* ChannelHandler 是给不同类型的事件调用
* 应用程序实现或扩展 ChannelHandler 挂接到事件生命周期和提供自定义应用逻辑。

###引导服务器

了解到业务核心处理逻辑 EchoServerHandler 后，下面要引导服务器自身了。

* 监听和接收进来的连接请求
* 配置 Channel 来通知一个关于入站消息的 EchoServerHandler 实例

*Transport(传输）*

*在本节中，你会遇到“transport(传输）”一词。在网络的多层视图协议里面，传输层提供了用于端至端或主机到主机的通信服务。互联网通信的基础是 TCP 传输。当我们使用术语“NIO transport”我们指的是一个传输的实现，它是大多等同于 TCP ，除了一些由 Java NIO 的实现提供了服务器端的性能增强。Transport 详细在第4章中讨论。*

Listing 2.3 EchoServer
	
	public class EchoServer {
	
	    private final int port;
	        
	    public EchoServer(int port) {
	        this.port = port;
	    }
		    public static void main(String[] args) throws Exception {
	        if (args.length != 1) {
	            System.err.println(
	                    "Usage: " + EchoServer.class.getSimpleName() +
	                    " <port>");
	            return;
	        }
	        int port = Integer.parseInt(args[0]);		//1
	        new EchoServer(port).start();				//2
	    }

	    public void start() throws Exception {
	        NioEventLoopGroup group = new NioEventLoopGroup(); //3
	        try {
	            ServerBootstrap b = new ServerBootstrap();
	            b.group(group)								//4
	             .channel(NioServerSocketChannel.class)		//5
	             .localAddress(new InetSocketAddress(port))	//6
	             .childHandler(new ChannelInitializer<SocketChannel>() { //7
	                 @Override
	                 public void initChannel(SocketChannel ch) 
	                     throws Exception {
	                     ch.pipeline().addLast(
	                             new EchoServerHandler());
	                 }
	             });
	
	            ChannelFuture f = b.bind().sync();			//8
	            System.out.println(EchoServer.class.getName() + " started and listen on " + f.channel().localAddress());
	            f.channel().closeFuture().sync();			//9
	        } finally {
	            group.shutdownGracefully().sync();			//10
	        }
	    }
	
	}

1.设置端口值（抛出一个 NumberFormatException 如果该端口参数的格式不正确）

2.呼叫服务器的 start() 方法

3.创建 EventLoopGroup

4.创建 ServerBootstrap

5.指定使用 NIO 的传输 Channel

6.设置 socket 地址使用所选的端口

7.添加 EchoServerHandler 到 Channel 的 ChannelPipeline

8.绑定的服务器;sync 等待服务器关闭

9.关闭 channel 和 块，直到它被关闭

10.关闭 EventLoopGroup，释放所有资源。

在这个例子中，代码创建 ServerBootstrap 实例（步骤4）。由于我们使用在 NIO 传输，我们已指定 NioEventLoopGroup（3）接受和处理新连接，指定 NioServerSocketChannel（5）为信道类型。在此之后，我们设置本地地址是 InetSocketAddress 与所选择的端口（6）如。服务器将绑定到此地址来监听新的连接请求。

第7步是关键：在这里我们使用一个特殊的类，ChannelInitializer 。当一个新的连接被接受，一个新的子 Channel 将被创建， ChannelInitializer 会添加我们EchoServerHandler 的实例到 Channel 的 ChannelPipeline。正如我们如前所述，如果有入站信息，这个处理器将被通知。

虽然 NIO 是可扩展性，但它的正确配置是不简单的。特别是多线程，要正确处理也非易事。幸运的是，Netty 的设计封装了大部分复杂性，尤其是通过抽象，例如 EventLoopGroup，SocketChannel 和 ChannelInitializer，其中每一个将在更详细地在第3章中讨论。

在步骤8，我们绑定的服务器，等待绑定完成。 （调用 sync() 的原因是当前线程阻塞）在第9步的应用程序将等待服务器 Channel 关闭（因为我们 在 Channel 的 CloseFuture 上调用 sync()）。现在，我们可以关闭下 EventLoopGroup 并释放所有资源，包括所有创建的线程（10）。

NIO 用于在本实施例，因为它是目前最广泛使用的传输，归功于它的可扩展性和彻底的不同步。但不同的传输的实现是也是可能的。例如，如果本实施例中使用的 OIO 传输，我们将指定
OioServerSocketChannel 和 OioEventLoopGroup。 Netty 的架构，包括更关于传输信息，将包含在第4章。在此期间，让我们回顾下在服务器上执行，我们只研究重要步骤。

服务器的主代码组件是

* EchoServerHandler 实现了的业务逻辑
* 在 main() 方法，引导了服务器

执行后者所需的步骤是：
* 创建 ServerBootstrap 实例来引导服务器并随后绑定
* 创建并分配一个 NioEventLoopGroup 实例来处理事件的处理，如接受新的连接和读/写数据。
* 指定本地 InetSocketAddress 给服务器绑定
* 通过 EchoServerHandler 实例给每一个新的 Channel 初始化
* 最后调用 ServerBootstrap.bind() 绑定服务器

这样服务器初始化完成，可以被使用了。

