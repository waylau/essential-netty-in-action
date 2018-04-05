案例研究:Transport 的迁移
======

为了让你想象 Transport 如何工作，我会从一个简单的应用程序开始，这个应用程序什么都不做，只是接受客户端连接并发送“Hi!”字符串消息到客户端，发送完了就断开连接。

###没有用 Netty 实现 I/O 和 NIO  

我们将不用 Netty 实现只用 JDK API 来实现 I/O 和 NIO。下面这个例子，是使用阻塞 IO 实现的例子：

Listing 4.1 Blocking networking without Netty
	
	public class PlainOioServer {
	
	    public void serve(int port) throws IOException {
	        final ServerSocket socket = new ServerSocket(port); 	//1
	        try {
	            for (;;) {
	                final Socket clientSocket = socket.accept();	//2
	                System.out.println("Accepted connection from " + clientSocket);
	
	                new Thread(new Runnable() {						//3
	                    @Override
	                    public void run() {
	                        OutputStream out;
	                        try {
	                            out = clientSocket.getOutputStream();
	                            out.write("Hi!\r\n".getBytes(Charset.forName("UTF-8")));							//4
	                            out.flush();
	                            clientSocket.close();				//5
	
	                        } catch (IOException e) {
	                            e.printStackTrace();
	                            try {
	                                clientSocket.close();
	                            } catch (IOException ex) {
	                                // ignore on close
	                            }
	                        }
	                    }
	                }).start();										//6
	            }
	        } catch (IOException e) {
	            e.printStackTrace();
	        }
	    }
	}

1.绑定服务器到指定的端口。

2.接受一个连接。

3.创建一个新的线程来处理连接。

4.将消息发送到连接的客户端。

5.一旦消息被写入和刷新时就 关闭连接。

6.启动线程。

上面的方式可以工作正常，但是这种阻塞模式在大连接数的情况就会有很严重的问题，如客户端连接超时，服务器响应严重延迟，性能无法扩展。为了解决这种情况，我们可以使用异步网络处理所有的并发连接，但问题在于 NIO 和 OIO 的 API 是完全不同的，所以一个用OIO开发的网络应用程序想要使用NIO重构代码几乎是重新开发。

下面代码是使用 NIO 实现的例子：

Listing 4.2 Asynchronous networking without Netty

	public class PlainNioServer {
	    public void serve(int port) throws IOException {
	        ServerSocketChannel serverChannel = ServerSocketChannel.open();
	        serverChannel.configureBlocking(false);
	        ServerSocket ss = serverChannel.socket();
	        InetSocketAddress address = new InetSocketAddress(port);
	        ss.bind(address);											//1
	        Selector selector = Selector.open();						//2
	        serverChannel.register(selector, SelectionKey.OP_ACCEPT);	//3
	        final ByteBuffer msg = ByteBuffer.wrap("Hi!\r\n".getBytes());
	        for (;;) {
	            try {
	                selector.select();									//4
	            } catch (IOException ex) {
	                ex.printStackTrace();
	                // handle exception
	                break;
	            }
	            Set<SelectionKey> readyKeys = selector.selectedKeys();	//5
	            Iterator<SelectionKey> iterator = readyKeys.iterator();
	            while (iterator.hasNext()) {
	                SelectionKey key = iterator.next();
	                iterator.remove();
	                try {
	                    if (key.isAcceptable()) {				//6
	                        ServerSocketChannel server =
	                                (ServerSocketChannel)key.channel();
	                        SocketChannel client = server.accept();
	                        client.configureBlocking(false);
	                        client.register(selector, SelectionKey.OP_WRITE |
	                                SelectionKey.OP_READ, msg.duplicate());	//7
	                        System.out.println(
	                                "Accepted connection from " + client);
	                    }
	                    if (key.isWritable()) {				//8
	                        SocketChannel client =
	                                (SocketChannel)key.channel();
	                        ByteBuffer buffer =
	                                (ByteBuffer)key.attachment();
	                        while (buffer.hasRemaining()) {
	                            if (client.write(buffer) == 0) {		//9
	                                break;
	                            }
	                        }
	                        client.close();					//10
	                    }
	                } catch (IOException ex) {
	                    key.cancel();
	                    try {
	                        key.channel().close();
	                    } catch (IOException cex) {
	                        // 在关闭时忽略
	                    }
	                }
	            }
	        }
	    }
	}
	
1.绑定服务器到制定端口

2.打开 selector 处理 channel

3.注册  ServerSocket 到 ServerSocket ，并指定这是专门意接受
连接。

4.等待新的事件来处理。这将阻塞，直到一个事件是传入。

5.从收到的所有事件中 获取 SelectionKey 实例。

6.检查该事件是一个新的连接准备好接受。

7.接受客户端，并用 selector 进行注册。

8.检查 socket 是否准备好写数据。

9.将数据写入到所连接的客户端。如果网络饱和，连接是可写的，那么这个循环将写入数据，直到该缓冲区是空的。

10.关闭连接。

如你所见，即使它们实现的功能是一样，但是代码完全不同。下面我们将用Netty 来实现相同的功能。

###采用 Netty 实现 I/O 和 NIO  

下面代码是使用Netty作为网络框架编写的一个阻塞 IO 例子：

Listing 4.3 Blocking networking with Netty
	
	public class NettyOioServer {
	
	    public void server(int port) throws Exception {
	        final ByteBuf buf = Unpooled.unreleasableBuffer(
	                Unpooled.copiedBuffer("Hi!\r\n", Charset.forName("UTF-8")));
	        EventLoopGroup group = new OioEventLoopGroup();
	        try {
	            ServerBootstrap b = new ServerBootstrap();		//1
	
	            b.group(group)									//2
	             .channel(OioServerSocketChannel.class)
	             .localAddress(new InetSocketAddress(port))
	             .childHandler(new ChannelInitializer<SocketChannel>() {//3
	                 @Override
	                 public void initChannel(SocketChannel ch) 
	                     throws Exception {
	                     ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {			//4
	                         @Override
	                         public void channelActive(ChannelHandlerContext ctx) throws Exception {
	                             ctx.writeAndFlush(buf.duplicate()).addListener(ChannelFutureListener.CLOSE);//5
	                         }
	                     });
	                 }
	             });
	            ChannelFuture f = b.bind().sync();  //6
	            f.channel().closeFuture().sync();
	        } finally {
	            group.shutdownGracefully().sync();		//7
	        }
	    }
	}

1.创建一个 ServerBootstrap

2.使用 OioEventLoopGroup 允许阻塞模式

3.指定 ChannelInitializer 将给每个接受的连接调用

4.添加的 ChannelHandler 拦截事件，并允许他们作出反应

5.写信息到客户端，并添加 ChannelFutureListener 当一旦消息写入就关闭连接

6.绑定服务器来接受连接

7.释放所有资源

下面代码是使用 Netty NIO 实现。

###Netty NIO 版本

下面是 Netty NIO 的代码，只是改变了一行代码，就从 OIO 传输 切换到了 NIO。

Listing 4.4 Asynchronous networking with Netty

	public class NettyNioServer {
	
	    public void server(int port) throws Exception {
	        final ByteBuf buf = Unpooled.unreleasableBuffer(
	                Unpooled.copiedBuffer("Hi!\r\n", Charset.forName("UTF-8")));
	        NioEventLoopGroup bossGroup = new NioEventLoopGroup(1);
		NioEventLoopGroup workerGroup = new NioEventLoopGroup();
	        try {
	            ServerBootstrap b = new ServerBootstrap();	//1
	            b.group(bossGroup, workerGroup)   //2
	             .channel(NioServerSocketChannel.class)
	             .localAddress(new InetSocketAddress(port))
	             .childHandler(new ChannelInitializer<SocketChannel>() {	//3
	                 @Override
	                 public void initChannel(SocketChannel ch) 
	                     throws Exception {
	                     ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {	//4
	                         @Override
	                         public void channelActive(ChannelHandlerContext ctx) throws Exception {
	                             ctx.writeAndFlush(buf.duplicate())				//5
									.addListener(ChannelFutureListener.CLOSE);
	                         }
	                     });
	                 }
	             });
	            ChannelFuture f = b.bind().sync();					//6
	            f.channel().closeFuture().sync();
	        } finally {
	            bossGroup.shutdownGracefully().sync();					//7
		    workerGroup.shutdownGracefully().sync();
	        }
	    }
	}


1.创建一个 ServerBootstrap

2.使用 NioEventLoopGroup 允许非阻塞模式（NIO）

3.指定 ChannelInitializer 将给每个接受的连接调用

4.添加的 ChannelInboundHandlerAdapter() 接收事件并进行处理

5.写信息到客户端，并添加 ChannelFutureListener 当一旦消息写入就关闭连接

6.绑定服务器来接受连接

7.释放所有资源

因为 Netty 使用相同的 API 来实现每个传输，它并不关心你使用什么来实现。Netty 通过操作接口Channel 、ChannelPipeline 和 ChannelHandler来实现。

现在你了解到了用 基于 Netty 传输的好处。下面就来看下传输的 API.

