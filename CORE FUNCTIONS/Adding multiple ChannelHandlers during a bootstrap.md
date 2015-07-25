在一个引导中添加多个 ChannelHandler
====

在所有的例子代码中，我们在引导过程中通过 handler() 或childHandler() 都只添加了一个 ChannelHandler 实例，对于简单的程序可能足够，但是对于复杂的程序则无法满足需求。例如，某个程序必须支持多个协议，如 HTTP、WebSocket。若在一个 ChannelHandle r中处理这些协议将导致一个庞大而复杂的 ChannelHandler。Netty 通过添加多个 ChannelHandler，从而使每个 ChannelHandler 分工明确，结构清晰。
    
Netty 的一个优势是可以在 ChannelPipeline 中堆叠很多ChannelHandler 并且可以最大程度的重用代码。如何添加多个ChannelHandler 呢？Netty 提供 ChannelInitializer 抽象类用来初始化 ChannelPipeline 中的 ChannelHandler。ChannelInitializer是一个特殊的 ChannelHandler，通道被注册到 EventLoop 后就会调用ChannelInitializer，并允许将 ChannelHandler 添加到CHannelPipeline；完成初始化通道后，这个特殊的 ChannelHandler 初始化器会从 ChannelPipeline 中自动删除。
    
听起来很复杂，其实很简单，看下面代码：

Listing 9.6 Bootstrap and using ChannelInitializer


    ServerBootstrap bootstrap = new ServerBootstrap();//1
    bootstrap.group(new NioEventLoopGroup(), new NioEventLoopGroup())  //2
		.channel(NioServerSocketChannel.class)  //3
        .childHandler(new ChannelInitializerImpl()); //4
    ChannelFuture future = bootstrap.bind(new InetSocketAddress(8080));  //5
    future.sync();


    final class ChannelInitializerImpl extends ChannelInitializer<Channel> {  //6
        @Override
        protected void initChannel(Channel ch) throws Exception {
            ChannelPipeline pipeline = ch.pipeline(); //7
            pipeline.addLast(new HttpClientCodec());
            pipeline.addLast(new HttpObjectAggregator(Integer.MAX_VALUE));

        }
    }

1. 创建一个新的 ServerBootstrap 来创建和绑定新的 Channel
2. 指定 EventLoopGroups 从 ServerChannel 和接收到的管道来注册并获取 EventLoops 
3. 指定 Channel 类来使用
4. 设置处理器用于处理接收到的管道的 I/O 和数据
5. 通过配置的引导来绑定管道
6. ChannelInitializer 负责设置 ChannelPipeline
7. 实现 initChannel() 来添加需要的处理器到 ChannelPipeline。一旦完成了这方法 ChannelInitializer 将会从 ChannelPipeline  删除自身。

通过 ChannelInitializer, Netty 允许你添加你程序所需的多个 ChannelHandler 到 ChannelPipeline