解码分隔符和基于长度的协议
====

使用 Netty 时会遇到需要解码以分隔符和长度为基础的协议，本节讲解Netty 如何解码这些协议。

### 分隔符协议
        
经常需要处理分隔符协议或创建基于它们的协议，例如[SMTP](http://www.ietf.org/rfc/rfc2821.txt)、[POP3](http://www.ietf.org/rfc/rfc1939.txt)、[IMAP](http://tools.ietf.org/html/rfc3501)、[Telnet](http://tools.ietf.org/search/rfc854)等等。Netty 附带的解码器可以很容易的提取一些序列分隔：

Table 8.5 Decoders for handling delimited and length-based protocols

名称 | 描述
-----|----
DelimiterBasedFrameDecoder | 接收ByteBuf由一个或多个分隔符拆分，如NUL或换行符
LineBasedFrameDecoder| 接收ByteBuf以分割线结束，如"\n"和"\r\n"

下图显示了使用"\r\n"分隔符的处理：

![](../images/Figure 8.5 Handling delimited frames.jpg)

1. 字节流
2. 第一帧
3. 第二帧

Figure 8.5 Handling delimited frames

下面展示了如何用 LineBasedFrameDecoder 处理

Listing 8.8 Handling line-delimited frames
    
    public class LineBasedHandlerInitializer extends ChannelInitializer<Channel> {
    
        @Override
        protected void initChannel(Channel ch) throws Exception {
            ChannelPipeline pipeline = ch.pipeline();
            pipeline.addLast(new LineBasedFrameDecoder(65 * 1024));   //1
            pipeline.addLast(new FrameHandler());  //2
        }
    
        public static final class FrameHandler extends SimpleChannelInboundHandler<ByteBuf> {
            @Override
            public void channelRead0(ChannelHandlerContext ctx, ByteBuf msg) throws Exception {  //3
                // Do something with the frame
            }
        }
    }

1. 添加一个 LineBasedFrameDecoder 用于提取帧并把数据包转发到下一个管道中的处理程序,在这种情况下就是 FrameHandler  
2. 添加 FrameHandler 用于接收帧
3. 每次调用都需要传递一个单帧的内容

使用 DelimiterBasedFrameDecoder 可以方便处理特定分隔符作为数据结构体的这类情况。如下：

* 传入的数据流是一系列的帧，每个由换行（“\n”）分隔
* 每帧包括一系列项目，每个由单个空格字符分隔
* 一帧的内容代表一个“命令”：一个名字后跟一些变量参数

清单8.9中显示了的实现的方式。定义以下类：

* 类 Cmd - 存储帧的内容，其中一个 ByteBuf 用于存名字，另外一个存参数
* 类 CmdDecoder - 从重写方法 decode() 中检索一行，并从其内容中构建一个 Cmd 的实例
* 类 CmdHandler - 从 CmdDecoder 接收解码 Cmd 对象和对它的一些处理。

所以关键的解码器是扩展了 LineBasedFrameDecoder

Listing 8.9 Decoder for the command and the handler

	public class CmdHandlerInitializer extends ChannelInitializer<Channel> {
	
	    @Override
	    protected void initChannel(Channel ch) throws Exception {
	        ChannelPipeline pipeline = ch.pipeline();
	        pipeline.addLast(new CmdDecoder(65 * 1024));//1
	        pipeline.addLast(new CmdHandler()); //2
	    }
	
	    public static final class Cmd { //3
	        private final ByteBuf name;
	        private final ByteBuf args;
	
	        public Cmd(ByteBuf name, ByteBuf args) {
	            this.name = name;
	            this.args = args;
	        }
	
	        public ByteBuf name() {
	            return name;
	        }
	
	        public ByteBuf args() {
	            return args;
	        }
	    }
	
	    public static final class CmdDecoder extends LineBasedFrameDecoder {
	        public CmdDecoder(int maxLength) {
	            super(maxLength);
	        }
	
	        @Override
	        protected Object decode(ChannelHandlerContext ctx, ByteBuf buffer) throws Exception {
	            ByteBuf frame =  (ByteBuf) super.decode(ctx, buffer); //4
	            if (frame == null) {
	                return null; //5
	            }
	            int index = frame.indexOf(frame.readerIndex(), frame.writerIndex(), (byte) ' ');  //6
	            return new Cmd(frame.slice(frame.readerIndex(), index), frame.slice(index +1, frame.writerIndex())); //7
	        }
	    }
	
	    public static final class CmdHandler extends SimpleChannelInboundHandler<Cmd> {
	        @Override
	        public void channelRead0(ChannelHandlerContext ctx, Cmd msg) throws Exception {
	            // Do something with the command  //8
	        }
	    }
	}

1. 添加一个 CmdDecoder 到管道；将提取 Cmd 对象和转发到在管道中的下一个处理器
2. 添加 CmdHandler 将接收和处理 Cmd 对象
3. 命令也是 POJO
4. super.decode() 通过结束分隔从 ByteBuf 提取帧
5. frame 是空时，则返回 null
6. 找到第一个空字符的索引。首先是它的命令名；接下来是参数的顺序
7. 从帧先于索引以及它之后的片段中实例化一个新的 Cmd 对象
8. 处理通过管道的 Cmd 对象