写一个 echo 客户端
=====

客户端要做的是：

* 连接服务器
* 发送信息
* 发送的每个信息，等待和接收从服务器返回的同样的信息
* 关闭连接

###用 ChannelHandler 实现客户端逻辑

跟写服务器一样，我们提供 ChannelInboundHandler 来处理数据。下面例子，我们用 SimpleChannelInboundHandler 来处理所有的任务，需要覆盖三个方法：

* channelActive() - 服务器的连接被建立后调用
* channelRead0() - 数据后从服务器接收到调用
* exceptionCaught() - 捕获一个异常时调用

Listing 2.4 ChannelHandler for the client

	@Sharable								//1
	public class EchoClientHandler extends
	        SimpleChannelInboundHandler<ByteBuf> {
	
	    @Override
	    public void channelActive(ChannelHandlerContext ctx) {
	        ctx.writeAndFlush(Unpooled.copiedBuffer("Netty rocks!", //2
			CharsetUtil.UTF_8));
	    }
	
	    @Override
	    public void channelRead0(ChannelHandlerContext ctx,
	        ByteBuf in) {
	        System.out.println("Client received: " + in.toString(CharsetUtil.UTF_8));	//3
	    }
	
	    @Override
	    public void exceptionCaught(ChannelHandlerContext ctx,
	        Throwable cause) {					//4
	        cause.printStackTrace();
	        ctx.close();
	    }
	}

1.`@Sharable`标记这个类的实例可以在 channel 里共享

2.当被通知该 channel 是活动的时候就发送信息

3.记录接收到的消息

4.记录日志错误并关闭 channel

建立连接后该 channelActive() 方法被调用一次。逻辑很简单：一旦建立了连接，字节序列被发送到服务器。该消息的内容并不重要;在这里，我们使用了 Netty 编码字符串 “Netty rocks!” 通过覆盖这种方法，我们确保东西被尽快写入到服务器。

接下来，我们覆盖方法 channelRead0()。这种方法会在接收到数据时被调用。注意，由服务器所发送的消息可以以块的形式被接收。即，当服务器发送 5 个字节是不是保证所有的 5 个字节会立刻收到 - 即使是只有 5 个字节，channelRead0() 方法可被调用两次，第一次用一个ByteBuf（Netty的字节容器）装载3个字节和第二次一个 ByteBuf 装载 2 个字节。唯一要保证的是，该字节将按照它们发送的顺序分别被接收。 （注意，这是真实的，只有面向流的协议如TCP）。

第三个方法重写是 exceptionCaught()。正如在 EchoServerHandler
（清单2.2），所述的记录 Throwable 并且关闭通道，在这种情况下终止
连接到服务器。

*SimpleChannelInboundHandler vs. ChannelInboundHandler*

*何时用这两个要看具体业务的需要。在客户端，当 channelRead0() 完成，我们已经拿到的入站的信息。当方法返回时，SimpleChannelInboundHandler 会小心的释放对 ByteBuf（保存信息） 的引用。而在 EchoServerHandler,我们需要将入站的信息返回给发送者，由于 write() 是异步的，在 channelRead() 返回时，可能还没有完成。所以，我们使用 ChannelInboundHandlerAdapter,无需释放信息。最后在 channelReadComplete() 我们调用 ctxWriteAndFlush() 来释放信息。详见第5、6章*

###引导客户端

客户端引导需要 host 、port 两个参数连接服务器。

Listing 2.5 Main class for the client
	
	public class EchoClient {
	
	    private final String host;
	    private final int port;
	
	    public EchoClient(String host, int port) {
	        this.host = host;
	        this.port = port;
	    }
	
	    public void start() throws Exception {
	        EventLoopGroup group = new NioEventLoopGroup();
	        try {
	            Bootstrap b = new Bootstrap();				//1
	            b.group(group)								//2
	             .channel(NioSocketChannel.class)			//3
	             .remoteAddress(new InetSocketAddress(host, port))	//4
	             .handler(new ChannelInitializer<SocketChannel>() {	//5
	                 @Override
	                 public void initChannel(SocketChannel ch) 
	                     throws Exception {
	                     ch.pipeline().addLast(
	                             new EchoClientHandler());
	                 }
	             });
	
	            ChannelFuture f = b.connect().sync();		//6
	
	            f.channel().closeFuture().sync();			//7
	        } finally {
	            group.shutdownGracefully().sync();			//8
	        }
	    }
	
	    public static void main(String[] args) throws Exception {
	        if (args.length != 2) {
	            System.err.println(
	                    "Usage: " + EchoClient.class.getSimpleName() +
	                    " <host> <port>");
	            return;
	        }
	
	        final String host = args[0];
	        final int port = Integer.parseInt(args[1]);
	
	        new EchoClient(host, port).start();
	    }
	}

1.创建 Bootstrap

2.指定 EventLoopGroup 来处理客户端事件。由于我们使用 NIO 传输，所以用到了 NioEventLoopGroup 的实现

3.使用的 channel 类型是一个用于 NIO 传输

4.设置服务器的 InetSocketAddress

5.当建立一个连接和一个新的通道时，创建添加到 EchoClientHandler 实例
到 channel pipeline 

6.连接到远程;等待连接完成

7.阻塞直到 Channel 关闭

8.调用 shutdownGracefully() 来关闭线程池和释放所有资源

与以前一样，在这里使用了 NIO 传输。请注意，您可以在 客户端和服务器 使用不同的传输
，例如 NIO 在服务器端和 OIO 客户端。在第四章中，我们将研究一些具体的因素和情况，这将导致
您可以选择一种传输，而不是另一种。

让我们回顾一下我们在本节所介绍的要点

* 一个 Bootstrap 被创建来初始化客户端
* 一个 NioEventLoopGroup 实例被分配给处理该事件的处理，这包括创建新的连接和处理入站和出站数据
* 一个 InetSocketAddress 为连接到服务器而创建
* 一个 EchoClientHandler 将被安装在 pipeline 当连接完成时
* 之后 Bootstrap.connect（）被调用连接到远程的 - 本例就是 echo(回声)服务器。

