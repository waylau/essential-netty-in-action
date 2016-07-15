Netty 编码器和解码器
====

Netty 的是一个复杂和先进的框架,但它并不玄幻。当我们请求一些设置了  key 的给定值时,我们知道 Request 类的一个实例被创建来代表这个请求。但 Netty 并不知道 Request 对象是如何转成 Memcached 所期望的。Memcached 所期望的是字节序列；忽略使用的协议，数据在网络上传输永远是字节序列。

将 Request 对象转为 Memcached 所需的字节序列，Netty 需要用 MemcachedRequest 来编码成另外一种格式。这里所说的另外一种格式不单单是从对象转为字节，也可以是从对象转为对象，或者是从对象转为字符串等。编码器的内容可以详见第七章。

Netty 提供了一个抽象类称为 MessageToByteEncoder。它提供了一个抽象方法,将一条消息(在本例中我们  MemcachedRequest 对象)转为字节。你显示什么信息实现通过使用 Java 泛型可以处理;例如 , MessageToByteEncoder<MemcachedRequest> 说这个编码器要编码的对象类型是 MemcachedRequest

*MessageToByteEncoder 和  Java 泛型*

*使用 MessageToByteEncoder 可以绑定特定的参数类型。如果你有多个不同的消息类型，在相同的编码器里，也可以使用MessageToByteEncoder<Object>，注意检查消息的类型即可*

这也适用于解码器,除了解码器将一系列字节转换回一个对象。
这个 Netty 的提供了 ByteToMessageDecoder 类,而不是提供一个编码方法用来实现解码。在接下来的两个部分你看看如何实现一个 Memcached 解码器和编码器。在你做之前,应该意识到在使用 Netty 时，你不总是需要自己提供编码器和解码器。自所以现在这么做是因为 Netty 没有对 Memcached 内置支持。而 HTTP 以及其他标准的协议，Netty 已经是提供的了。

*编码器和解码器*

*记住,编码器处理出站，而解码器处理入站。这基本上意味着编码器将编码数据,写入远端。解码器将从远端读取处理数据。重要的是要记住,出站和入站是两个不同的方向。*

请注意,为了程序简单，我们的编码器和解码器不检查任何值的最大大小。在实际实现中你需要一些验证检查,如果检测到违反协议，则使用 EncoderException 或 DecoderException(或一个子类)。


### 实现 Memcached 编码器

本节我们将简要介绍编码器的实现。正如我们提到的,编码器负责编码消息为字节序列。这些字节可以通过网络发送到远端。为了发送请求，我们首先创建 MemcachedRequest 类,稍后编码器实现会编码为一系列字节。下面的清单显示了我们的 MemcachedRequest 类

Listing 14.1 Implementation of a Memcached request

	public class MemcachedRequest { //1
	    private static final Random rand = new Random();
	    private final int magic = 0x80;//fixed so hard coded
	    private final byte opCode; //the operation e.g. set or get
	    private final String key; //the key to delete, get or set
	    private final int flags = 0xdeadbeef; //random
	    private final int expires; //0 = item never expires
	    private final String body; //if opCode is set, the value
	    private final int id = rand.nextInt(); //Opaque
	    private final long cas = 0; //data version check...not used
	    private final boolean hasExtras; //not all ops have extras
	
	    public MemcachedRequest(byte opcode, String key, String value) {
	        this.opCode = opcode;
	        this.key = key;
	        this.body = value == null ? "" : value;
	        this.expires = 0;
	        //only set command has extras in our example
	        hasExtras = opcode == Opcode.SET;
	    }
	
	    public MemcachedRequest(byte opCode, String key) {
	        this(opCode, key, null);
	    }
	
	    public int magic() { //2
	        return magic;
	    }
	
	    public int opCode() {  //3
	        return opCode;
	    }
	
	    public String key() {  //4
	        return key;
	    }
	
	    public int flags() {  //5
	        return flags;
	    }
	
	    public int expires() {  //6
	        return expires;
	    }
	
	    public String body() {  //7
	        return body;
	    }
	
	    public int id() {  //8
	        return id;
	    }
	
	    public long cas() {  //9
	        return cas;
	    }
	
	    public boolean hasExtras() {  //10
	        return hasExtras;
	    }
	}

1. 这个类将会发送请求到 Memcached server
2. 幻数，它可以用来标记文件或者协议的格式
3. opCode,反应了响应的操作已经创建了
4. 执行操作的 key
5. 使用的额外的 flag
6. 表明到期时间
7. body
8. 请求的 id。这个id将在响应中回显。
9. compare-and-check 的值
10. 如果有额外的使用，将返回 true

你如果想实现 Memcached 的其余部分协议,你只需要将 client.op*(op * 任何新的操作添加)转换为其中一个方法请求。我们需要两个更多的支持类,在下一个清单所示

Listing 14.2 Possible Memcached operation codes and response statuses
	
	public class Status {
		public static final short NO_ERROR = 0x0000;
		public static final short KEY_NOT_FOUND = 0x0001;
		public static final short KEY_EXISTS = 0x0002;
		public static final short VALUE_TOO_LARGE = 0x0003;
		public static final short INVALID_ARGUMENTS = 0x0004;
		public static final short ITEM_NOT_STORED = 0x0005;
		public static final short INC_DEC_NON_NUM_VAL = 0x0006;
	}
	public class Opcode {
		public static final byte GET = 0x00;
		public static final byte SET = 0x01;
		public static final byte DELETE = 0x04;
	}

一个 Opcode 告诉 Memcached 要执行哪些操作。每个操作都由一个字节表示。同样的,当 Memcached 响应一个请求,响应头中包含两个字节代表响应状态。状态和 Opcode 类表示这些 Memcached 的构造。这些操作码可以使用当你构建一个新的 MemcachedRequest 指定哪个行动应该由它引发的。

但现在可以集中精力在编码器上：

Listing 14.3 MemcachedRequestEncoder implementation

	public class MemcachedRequestEncoder extends
	        MessageToByteEncoder<MemcachedRequest> { //1
	    @Override
	    protected void encode(ChannelHandlerContext ctx, MemcachedRequest msg,
	                          ByteBuf out) throws Exception {  //2
	        byte[] key = msg.key().getBytes(CharsetUtil.UTF_8);
	        byte[] body = msg.body().getBytes(CharsetUtil.UTF_8);
	        //total size of the body = key size + content size + extras size   //3
	        int bodySize = key.length + body.length + (msg.hasExtras() ? 8 : 0);
	
	        //write magic byte  //4
	        out.writeByte(msg.magic());
	        //write opcode byte  //5
	        out.writeByte(msg.opCode());
	        //write key length (2 byte) //6
	        out.writeShort(key.length); //key length is max 2 bytes i.e. a Java short  //7
	        //write extras length (1 byte)
	        int extraSize =  msg.hasExtras() ? 0x08 : 0x0;
	        out.writeByte(extraSize);
	        //byte is the data type, not currently implemented in Memcached but required //8
	        out.writeByte(0);
	        //next two bytes are reserved, not currently implemented but are required  //9
	        out.writeShort(0);
	
	        //write total body length ( 4 bytes - 32 bit int)  //10
	        out.writeInt(bodySize);
	        //write opaque ( 4 bytes)  -  a 32 bit int that is returned in the response //11
	        out.writeInt(msg.id());
	
	        //write CAS ( 8 bytes)
	        out.writeLong(msg.cas());   //24 byte header finishes with the CAS  //12
	
	        if (msg.hasExtras()) {
	            //write extras (flags and expiry, 4 bytes each) - 8 bytes total  //13
	            out.writeInt(msg.flags());
	            out.writeInt(msg.expires());
	        }
	        //write key   //14
	        out.writeBytes(key);
	        //write value  //15
	        out.writeBytes(body);
	    }
	}
	
1. 该类是负责编码 MemachedRequest 为一系列字节
2. 转换的 key 和实际请求的 body 到字节数组
3. 计算 body 大小
4. 写幻数到 ByteBuf 字节
5. 写 opCode 作为字节
6. 写 key 长度z作为 short
7. 编写额外的长度作为字节
8. 写数据类型,这总是0,因为目前不是在 Memcached,但可用于使用
后来的版本
9. 为保留字节写为 short ,后面的 Memcached 版本可能使用
10. 写 body 的大小作为 long
11. 写 opaque 作为 int
12. 写 cas 作为 long。这个是头文件的最后部分，在 body 的开始
13. 编写额外的 flag 和到期时间为 int
14. 写 key 
15. 这个请求完成后 写 body。

总结，编码器 使用 Netty 的 ByteBuf 处理请求，编码 MemcachedRequest 成一套正确排序的字节。详细步骤为：


* 写幻数字节。
* 写 opcode 字节。
* 写 key 长度(2字节)。
* 写额外的长度(1字节)。
* 写数据类型(1字节)。
* 为保留字节写 null 字节(2字节)。
* 写 body 长度(4字节- 32位整数)。
* 写 opaque(4个字节,一个32位整数在响应中返回)。
* 写 CAS(8个字节)。
* 写 额外的(flag 和 到期,4字节)= 8个字节
* 写 key
* 写 值

无论你放入什么到输出缓冲区( 调用 ByteBuf) Netty 的将向服务器发送被写入请求。下一节将展示如何进行反向通过解码器工作。

### 实现 Memcached 解码器

将 MemcachedRequest 对象转为字节序列，Memcached 仅需将字节转到响应对象返回即可。

先见一个 POJO:

Listing 14.7 Implementation of a MemcachedResponse

	public class MemcachedResponse {  //1
	    private final byte magic;
	    private final byte opCode;
		private byte dataType;
	    private final short status;
	    private final int id;
	    private final long cas;
	    private final int flags;
	    private final int expires;
	    private final String key;
	    private final String data;
	
	    public MemcachedResponse(byte magic, byte opCode,
				byte dataType,                             short status, 
				int id, long cas,
	            int flags, int expires, String key, String data) {
	        this.magic = magic;
	        this.opCode = opCode;
			this.dataType = dataType;
	        this.status = status;
	        this.id = id;
	        this.cas = cas;
	        this.flags = flags;
	        this.expires = expires;
	        this.key = key;
	        this.data = data;
	    }
	
	    public byte magic() { //2
	        return magic;
	    }
	
	    public byte opCode() { //3
	        return opCode;
	    }

		public byte dataType() { //4
			return dataType;
		}

	    public short status() {  //5
	        return status;
	    }
	
	    public int id() {  //6
	        return id;
	    }
	
	    public long cas() {  //7
	        return cas;
	    }
	
	    public int flags() {  //8
	        return flags;
	    }
	
	    public int expires() { //9
	        return expires;
	    }
	
	    public String key() {  //10
	        return key;
	    }
	
	    public String data() {  //11
	        return data; 
	    }
	}

1. 该类,代表从 Memcached 服务器返回的响应
2. 幻数
3. opCode,这反映了创建操作的响应
4. 数据类型,这表明这个是基于二进制还是文本
5. 响应的状态,这表明如果请求是成功的
6. 惟一的 id
7. compare-and-set 值
8. 使用额外的 flag
9. 表示该值存储的一个有效期
10. 响应创建的 key 
11. 实际数据

下面为 MemcachedResponseDecoder， 使用了 ByteToMessageDecoder 基类，用于将 字节序列转为  MemcachedResponse

Listing 14.4 MemcachedResponseDecoder class

	public class MemcachedResponseDecoder extends ByteToMessageDecoder {  //1
	    private enum State {  //2
	        Header,
	        Body
	    }
	
	    private State state = State.Header;
	    private int totalBodySize;
	    private byte magic;
	    private byte opCode;
	    private short keyLength;
	    private byte extraLength;
	    private short status;
	    private int id;
	    private long cas;
	
	    @Override
	    protected void decode(ChannelHandlerContext ctx, ByteBuf in,
	                          List<Object> out) { 
	        switch (state) { //3
	            case Header:
	                if (in.readableBytes() < 24) {
	                    return;//response header is 24  bytes  //4
	                }
	                magic = in.readByte();  //5
	                opCode = in.readByte();
	                keyLength = in.readShort();
	                extraLength = in.readByte();
	                in.skipBytes(1);
	                status = in.readShort();
	                totalBodySize = in.readInt();
	                id = in.readInt(); //referred to in the protocol spec as opaque
	                cas = in.readLong();
	
	                state = State.Body;
	            case Body:
	                if (in.readableBytes() < totalBodySize) {
	                    return; //until we have the entire payload return  //6
	                }
	                int flags = 0, expires = 0;
	                int actualBodySize = totalBodySize;
	                if (extraLength > 0) {  //7
	                    flags = in.readInt();
	                    actualBodySize -= 4;
	                }
	                if (extraLength > 4) {  //8
	                    expires = in.readInt();
	                    actualBodySize -= 4;
	                }
	                String key = "";
	                if (keyLength > 0) {  //9
	                    ByteBuf keyBytes = in.readBytes(keyLength);
	                    key = keyBytes.toString(CharsetUtil.UTF_8);
	                    actualBodySize -= keyLength;
	                }
	                ByteBuf body = in.readBytes(actualBodySize);  //10
	                String data = body.toString(CharsetUtil.UTF_8);
	                out.add(new MemcachedResponse(  //1
	                        magic,
	                        opCode,
	                        status,
	                        id,
	                        cas,
	                        flags,
	                        expires,
	                        key,
	                        data
	                ));
	
	                state = State.Header;
	        }
	
	    }
	}
	
1. 类负责创建的 MemcachedResponse 读取字节
2. 代表当前解析状态,这意味着我们需要解析的头或 body
3. 根据解析状态切换
4. 如果不是至少24个字节是可读的,它不可能读整个头部,所以返回这里,等待再通知一次数据准备阅读
5. 阅读所有头的字段
6. 检查是否足够的数据是可读用来读取完整的响应的 body。长度是从头读取
7. 检查如果有任何额外的 flag 用于读，如果是这样做
8. 检查如果响应包含一个 expire 字段，有就读它
9. 检查响应是否包含一个 key ,有就读它
10. 读实际的 body 的 payload
11. 从前面读取字段和数据构造一个新的 MemachedResponse 

所以在实现发生了什么事?我们知道一个 Memcached 响应有24位头;我们不知道是否所有数据,响应将被包含在输入 ByteBuf ，当解码方法调用时。这是因为底层网络堆栈可能将数据分解成块。所以确保我们只解码当我们有足够的数据,这段代码检查是否可用可读的字节的数量至少是24。一旦我们有24个字节,我们可以确定整个消息有多大,因为这个信息包含在24位头。

当我们解码整个消息,我们创建一个 MemcachedResponse 并将其添加到输出列表。任何对象添加到该列表将被转发到下一个ChannelInboundHandler 在 ChannelPipeline,因此允许处理。

