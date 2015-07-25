引导 DatagramChannel
====

在前面引导代码示例使用 SocketChannel,这是基于 tcp 的。但引导也可以用于 UDP 等无连接协议。这个使用 Netty 的提供各种DatagramChannel 实现。唯一的区别是,你无需调用 connect() 但只是 bind(),如清单9.8所示

Listing 9.8 Using Bootstrap with DatagramChannel

	Bootstrap bootstrap = new Bootstrap(); //1
	bootstrap.group(new OioEventLoopGroup()).channel ( //2
		OioDatagramChannel.class)  //3
	        .handler(new SimpleChannelInboundHandler<DatagramPacket>(){ //4
	            @Override
	            public void channelRead0(ChannelHandlerContext ctx, DatagramPacket msg) throws Exception {
	                // Do something with the packet
	            }
	        });
	ChannelFuture future = bootstrap.bind(new InetSocketAddress(0));  //5
	future.addListener(new ChannelFutureListener() {
	    @Override
	    public void operationComplete(ChannelFuture channelFuture) throws Exception {
	        if (channelFuture.isSuccess()) {
	            System.out.println("Channel bound");
	        } else {
	            System.err.println("Bound attempt failed");
	            channelFuture.cause().printStackTrace();
	        }
	    }
	});


1. 新建 Bootstrap 用于创建和绑定到新的 datagram channel
2. 指定 EventLoopGroup 用来从和注册的管道中获取 EventLoop
3. 指定 Channel 类
4. 设置处理器来处理管道中的 I/O 和数据
5. 当协议是无连接时，调用 bind() 