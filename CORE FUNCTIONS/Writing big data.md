编写大型数据
====

由于网络的原因，如何有效的写大数据在异步框架是一个特殊的问题。因为写操作是非阻塞的，即便是在数据不能写出时,只是通知 ChannelFuture 完成了。当这种情况发生时,你必须停止写操作或面临内存耗尽的风险。所以写时,会产生大量的数据,我们需要做好准备来处理的这种情况下的缓慢的连接远端导致延迟释放内存的问题你。作为一个例子让我们考虑写一个文件的内容到网络。

在我们的讨论传输(见4.2节)时，我们提到了 NIO 的“zero-copy（零拷贝）”功能,消除移动一个文件的内容从文件系统到网络堆栈的复制步骤。所有这一切发生在 Netty 的核心,因此所有所需的应用程序代码是使用 interface FileRegion 的实现,在 Netty 的API 文档中定义如下为一个通过 Channel 支持 zero-copy 文件传输的文件区域。

下面演示了通过 zero-copy 将文件内容从 FileInputStream 创建 DefaultFileRegion 并写入  使用 Channel

Listing 8.11 Transferring file contents with FileRegion

	FileInputStream in = new FileInputStream(file); //1
    FileRegion region = new DefaultFileRegion(in.getChannel(), 0, file.length()); //2

    channel.writeAndFlush(region).addListener(new ChannelFutureListener() { //3
        @Override
        public void operationComplete(ChannelFuture future) throws Exception {
            if (!future.isSuccess()) {
                Throwable cause = future.cause(); //4
                // Do something
            }
        }
    });

1. 获取 FileInputStream
2. 创建一个新的 DefaultFileRegion 用于文件的完整长度
3. 发送 DefaultFileRegion 并且注册一个 ChannelFutureListener
4. 处理发送失败

只是看到的例子只适用于直接传输一个文件的内容,没有执行的数据应用程序的处理。在相反的情况下,将数据从文件系统复制到用户内存是必需的,您可以使用 ChunkedWriteHandler。这个类提供了支持异步写大数据流不引起高内存消耗。

这个关键是 interface ChunkedInput，实现如下：

名称 | 描述
-----|----
ChunkedFile | 当你使用平台不支持 zero-copy 或者你需要转换数据，从文件中一块一块的获取数据
ChunkedNioFile | 与 ChunkedFile 类似，处理使用了NIOFileChannel
ChunkedStream | 从 InputStream 中一块一块的转移内容
ChunkedNioStream | 从 ReadableByteChannel 中一块一块的转移内容

清单 8.12 演示了使用 ChunkedStream,实现在实践中最常用。
所示的类被实例化一个 File 和一个 SslContext。当 initChannel() 被调用来初始化显示的处理程序链的通道。

当通道激活时，WriteStreamHandler 从文件一块一块的写入数据作为ChunkedStream。最后将数据通过 SslHandler 加密后传播。

Listing 8.12 Transfer file content with FileRegion

    public class ChunkedWriteHandlerInitializer extends ChannelInitializer<Channel> {
        private final File file;
        private final SslContext sslCtx;
        
        public ChunkedWriteHandlerInitializer(File file, SslContext sslCtx) {
            this.file = file;
            this.sslCtx = sslCtx;
        }
    
        @Override
        protected void initChannel(Channel ch) throws Exception {
            ChannelPipeline pipeline = ch.pipeline();
            pipeline.addLast(new SslHandler(sslCtx.createEngine()); //1
            pipeline.addLast(new ChunkedWriteHandler());//2
            pipeline.addLast(new WriteStreamHandler());//3
        }
    
        public final class WriteStreamHandler extends ChannelInboundHandlerAdapter {  //4
    
            @Override
            public void channelActive(ChannelHandlerContext ctx) throws Exception {
                super.channelActive(ctx);
                ctx.writeAndFlush(new ChunkedStream(new FileInputStream(file)));
            }
        }
    }
    
    
1. 添加 SslHandler 到 ChannelPipeline.
2. 添加 ChunkedWriteHandler 用来处理作为 ChunkedInput 传进的数据
3. 当连接建立时，WriteStreamHandler 开始写文件的内容
4. 当连接建立时，channelActive() 触发使用 ChunkedInput 来写文件的内容 (插图显示了 FileInputStream;也可以使用任何 InputStream )

*ChunkedInput*
*所有被要求使用自己的 ChunkedInput 实现，是安装ChunkedWriteHandler 在管道中*

在本节中,我们讨论
* 如何采用zero-copy（零拷贝）功能高效地传输文件
* 如何使用 ChunkedWriteHandler 编写大型数据而避免 OutOfMemoryErrors 错误。

在下一节中我们将研究几种不同方法来序列化 POJO。