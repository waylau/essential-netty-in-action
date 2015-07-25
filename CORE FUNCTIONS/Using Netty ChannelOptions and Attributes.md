使用Netty 的 ChannelOption 和属性
====

比较麻烦的是创建通道后不得不手动配置每个通道，为了避免这种情况，Netty 提供了 ChannelOption 来帮助引导配置。这些选项会自动应用到引导创建的所有通道，可用的各种选项可以配置底层连接的详细信息，如通道“keep-alive(保持活跃)”或“timeout(超时)”的特性。

Netty 应用程序通常会与组织或公司其他的软件进行集成，在某些情况下，Netty 的组件如 Channel 在 Netty 正常生命周期外使用；
Netty 的提供了抽象 AttributeMap 集合,这是由 Netty　的管道和引导类,和　AttributeKey<T>，常见类用于插入和检索属性值。属性允许您安全的关联任何数据项与客户端和服务器的　Channel。

例如,考虑一个服务器应用程序跟踪用户和　Channel　之间的关系。这可以通过存储用户　ID　作为　Channel　的一个属性。类似的技术可以用来路由消息到基于用户　ID　或关闭基于用户活动的一个管道。

清单9.7展示了如何使用　ChannelOption　配置　Channel　和一个属性来存储一个整数值。

Listing 9.7 Using Attributes

	final AttributeKey<Integer> id = new AttributeKey<Integer>("ID");　//1
	
	Bootstrap bootstrap = new Bootstrap(); //2
	bootstrap.group(new NioEventLoopGroup()) //3
			.channel(NioSocketChannel.class) //4
	        .handler(new SimpleChannelInboundHandler<ByteBuf>() { //5
	            @Override
	            public void channelRegistered(ChannelHandlerContext ctx) throws Exception {
	               Integer idValue = ctx.channel().attr(id).get();  //6
	                // do something  with the idValue
	            }
	
	            @Override
	            protected void channelRead0(ChannelHandlerContext channelHandlerContext, ByteBuf byteBuf) throws Exception {
	                System.out.println("Reveived data");
	            }
	        });
	bootstrap.option(ChannelOption.SO_KEEPALIVE, true).option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000);   //7
	bootstrap.attr(id, 123456); //8

	ChannelFuture future = bootstrap.connect(new InetSocketAddress("www.manning.com", 80));   //9
	future.syncUninterruptibly();

1. 新建一个 AttributeKey 用来存储属性值
2. 新建 Bootstrap 用来创建客户端管道并连接他们
3. 指定 EventLoopGroups 从和接收到的管道来注册并获取 EventLoop
4. 指定 Channel 类
5. 设置处理器来处理管道的 I/O 和数据
6. 检索 AttributeKey 的属性及其值
7. 设置 ChannelOption 将会设置在管道在连接或者绑定
8. 存储 id 属性
9. 通过配置的 Bootstrap 来连接到远程主机