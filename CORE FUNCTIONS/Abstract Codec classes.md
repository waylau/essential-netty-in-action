抽象 Codec(编解码器)类
====

虽然我们一直把解码器和编码器作为不同的实体来讨论，但你有时可能会发现把入站和出站的数据和信息转换都放在同一个类中更实用。Netty的抽象编解码器类就是用于这个目的,他们把一些成对的解码器和编码器组合在一起，以此来提供对于字节和消息都相同的操作。(这些类实现了
ChannelInboundHandler 和 ChannelOutboundHandler )。

您可能想知道是否有时候使用单独的解码器和编码器会比使用这些组合类要好，最简单的答案是,紧密耦合的两个函数减少了他们的可重用性,但是把他们分开实现就会更容易扩展。当我们研究抽象编解码器类时，我们也会拿它和对应的独立的解码器和编码器做对比。

### ByteToMessageCodec 

我们需要解码字节到消息,也许是一个 POJO,然后转回来。ByteToMessageCodec 将为我们处理这个问题,因为它结合了ByteToMessageDecoder 和 MessageToByteEncoder。表7.5中列出的重要方法。

Table 7.5 ByteToMessageCodec API

方法名称 | 描述
--------|----
decode | This method is called as long as bytes are available to be consumed. It converts the inbound ByteBuf to the specified message format and forwards them to the next ChannelInboundHandler in the pipeline.
decodeLast | The default implementation of this method delegates to decode(). It is called only be called once, when the Channel goes inactive. For special handling it can be oerridden.
encode | This method is called for each message to be written through the ChannelPipeline. The encoded messages are contained in a ByteBuf which

什么会是一个好的 ByteToMessageCodec 用例?任何一个请求/响应协议都可能是,例如 SMTP。编解码器将读取入站字节并解码到一个自定义的消息类型 SmtpRequest 。当接收到一个 SmtpResponse 会产生,用于编码为字节进行传输。

###  MessageToMessageCodec 

7.3.2节中我们看到的一个例子使用 MessageToMessageEncoder 从一个消息格式转换到另一个地方。现在让我们看看 MessageToMessageCodec 是如何处理 单个类 的往返。

在进入细节之前,让我们看看表7.6中的重要方法。

Table 7.6 Methods of MessageToMessageCodec

方法名称 | 描述
--------|----
decode | This method is called with the inbound messages of the codec and decodes them to messages. Those messages are forwarded to the next ChannelInboundHandler in the ChannelPipeline
decodeLast | Default implementation delegates to decode().decodeLast will only be called one time, which is when the Channel goes inactive. If you need special handling here you may override decodeLast() to implement it.
encode | The encode method is called for each outbound message to be moved through the ChannelPipeline. The encoded messages are forwarded to the next ChannelOutboundHandler in the pipeline

MessageToMessageCodec 是一个参数化的类，定义如下：

	public abstract class MessageToMessageCodec<INBOUND,OUTBOUND>

上面所示的完整签名的方法都是这样的

	protected abstract void encode(ChannelHandlerContext ctx,
	OUTBOUND msg, List<Object> out)
	protected abstract void decode(ChannelHandlerContext ctx,
	INBOUND msg, List<Object> out)

encode() 处理出站消息类型 OUTBOUND 到 INBOUND，而 decode() 则相反。我们在哪里可能使用这样的编解码器?

在现实中,这是一个相当常见的用例,往往涉及两个来回转换的数据消息传递API 。这是常有的事,当我们不得不与遗留或专有的消息格式进行互操作。

如清单7.7所示这样的可能性。在这个例子中,WebSocketConvertHandler 是一个静态嵌套类，继承了参数为 WebSocketFrame（类型为 INBOUND）和 WebSocketFrame（类型为 OUTBOUND）的 MessageToMessageCode

Listing 7.7 MessageToMessageCodec

	public class WebSocketConvertHandler extends MessageToMessageCodec<WebSocketFrame, WebSocketConvertHandler.WebSocketFrame> {  //1
	
	    public static final WebSocketConvertHandler INSTANCE = new WebSocketConvertHandler();
	
	    @Override
	    protected void encode(ChannelHandlerContext ctx, WebSocketFrame msg, List<Object> out) throws Exception {   
	        ByteBuf payload = msg.getData().duplicate().retain();
	        switch (msg.getType()) {   //2
	            case BINARY:
	                out.add(new BinaryWebSocketFrame(payload));
	                break;
	            case TEXT:
	                out.add(new TextWebSocketFrame(payload));
	                break;
	            case CLOSE:
	                out.add(new CloseWebSocketFrame(true, 0, payload));
	                break;
	            case CONTINUATION:
	                out.add(new ContinuationWebSocketFrame(payload));
	                break;
	            case PONG:
	                out.add(new PongWebSocketFrame(payload));
	                break;
	            case PING:
	                out.add(new PingWebSocketFrame(payload));
	                break;
	            default:
	                throw new IllegalStateException("Unsupported websocket msg " + msg);
	        }
	    }
	
	    @Override
	    protected void decode(ChannelHandlerContext ctx, io.netty.handler.codec.http.websocketx.WebSocketFrame msg, List<Object> out) throws Exception {
	        if (msg instanceof BinaryWebSocketFrame) {  //3
	            out.add(new WebSocketFrame(WebSocketFrame.FrameType.BINARY, msg.content().copy()));
	        } else if (msg instanceof CloseWebSocketFrame) {
	            out.add(new WebSocketFrame(WebSocketFrame.FrameType.CLOSE, msg.content().copy()));
	        } else if (msg instanceof PingWebSocketFrame) {
	            out.add(new WebSocketFrame(WebSocketFrame.FrameType.PING, msg.content().copy()));
	        } else if (msg instanceof PongWebSocketFrame) {
	            out.add(new WebSocketFrame(WebSocketFrame.FrameType.PONG, msg.content().copy()));
	        } else if (msg instanceof TextWebSocketFrame) {
	            out.add(new WebSocketFrame(WebSocketFrame.FrameType.TEXT, msg.content().copy()));
	        } else if (msg instanceof ContinuationWebSocketFrame) {
	            out.add(new WebSocketFrame(WebSocketFrame.FrameType.CONTINUATION, msg.content().copy()));
	        } else {
	            throw new IllegalStateException("Unsupported websocket msg " + msg);
	        }
	    }
	
	    public static final class WebSocketFrame {  //4
	        public enum FrameType {		//5
	            BINARY,
	            CLOSE,
	            PING,
	            PONG,
	            TEXT,
	            CONTINUATION
	        }
	
	        private final FrameType type;
	        private final ByteBuf data;
	        public WebSocketFrame(FrameType type, ByteBuf data) {
	            this.type = type;
	            this.data = data;
	        }
	
	        public FrameType getType() {
	            return type;
	        }
	
	        public ByteBuf getData() {
	            return data;
	        }
	    }
	}
	
	
1. 编码 WebSocketFrame 消息转为 WebSocketFrame 消息
2. 检测 WebSocketFrame 的 FrameType 类型，并且创建一个新的响应的 FrameType 类型的 WebSocketFrame
3. 通过 instanceof 来检测正确的 FrameType
4. 自定义消息类型 WebSocketFrame
5. 枚举类明确了 WebSocketFrame 的类型

### CombinedChannelDuplexHandler 

如前所述,结合解码器和编码器在一起可能会牺牲可重用性。为了避免这种方式，并且部署一个解码器和编码器到 ChannelPipeline 作为逻辑单元而不失便利性。

关键是下面的类:

	public class CombinedChannelDuplexHandler<I extends ChannelInboundHandler,O extends ChannelOutboundHandler>

这个类是扩展 ChannelInboundHandler 和 ChannelOutboundHandler 参数化的类型。这提供了一个容器,单独的解码器和编码器类合作而无需直接扩展抽象的编解码器类。我们将在下面的例子说明这一点。首先查看 ByteToCharDecoder ，如清单7.8所示。

Listing 7.8 ByteToCharDecoder

	public class ByteToCharDecoder extends
	        ByteToMessageDecoder { //1
	
	    @Override
	    public void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out)
	            throws Exception {
	        if (in.readableBytes() >= 2) {  //2
	            out.add(in.readChar());
	        }
	    }
	}

1. 继承 ByteToMessageDecoder
2. 写 char 到 MessageBuf

decode() 方法从输入数据中提取两个字节,并将它们作为一个 char 写入 List 。(注意,实现扩展 ByteToMessageDecoder 因为它从 ByteBuf 读取字符。)

现在看一下清单7.9中,把字符转换为字节的编码器。

Listing 7.9 CharToByteEncoder

	public class CharToByteEncoder extends
	        MessageToByteEncoder<Character> { //1
	
	    @Override
	    public void encode(ChannelHandlerContext ctx, Character msg, ByteBuf out)
	            throws Exception {
	        out.writeChar(msg);   //2
	    }
	}

1. 继承 MessageToByteEncoder
2. 写 char 到 ByteBuf

这个实现继承自 MessageToByteEncoder 因为他需要编码 char 消息 到 ByteBuf。这将直接将字符串写为 ByteBuf。

现在我们有编码器和解码器，将他们组成一个编解码器。见下面的 CombinedChannelDuplexHandler.

Listing 7.10 CombinedByteCharCodec
	
	public class CombinedByteCharCodec extends CombinedChannelDuplexHandler<ByteToCharDecoder, CharToByteEncoder> {
	    public CombinedByteCharCodec() {
	        super(new ByteToCharDecoder(), new CharToByteEncoder());
	    }
	}

1. CombinedByteCharCodec 的参数是解码器和编码器的实现用于处理进站字节和出站消息
2. 传递 ByteToCharDecoder 和 CharToByteEncoder 实例到 super 构造函数来委托调用使他们结合起来。

正如你所看到的,它可能是用上述方式来使程序更简单、更灵活,而不是使用一个以上的编解码器类。它也可以归结到你个人喜好或风格。

