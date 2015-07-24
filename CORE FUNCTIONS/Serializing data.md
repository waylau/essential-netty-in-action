序列化数据
====

JDK 提供了 ObjectOutputStream 和 ObjectInputStream 通过网络将原始数据类型和 POJO 进行序列化和反序列化。API并不复杂,可以应用到任何对象,支持 java.io.Serializable 接口。但它也不是非常高效的。在本节中,我们将看到 Netty 所提供的。

### JDK 序列化

如果程序与端对端间的交互是使用 ObjectOutputStream 和
ObjectInputStream，并且主要面临的问题是兼容性，那么，
[JDK 序列化](http://docs.oracle.com/javase/7/docs/technotes/guides/serialization/) 是不错的选择。

表8.8列出了序列化类，Netty 提供了与 JDK 的互操作。

Table 8.8 JDK Serialization codecs

名称 | 描述
-----|----
CompatibleObjectDecoder | 该解码器使用 JDK 序列化，用于与非 Netty 进行互操作。
CompatibleObjectEncoder | 该编码器使用 JDK 序列化，用于与非 Netty 进行互操作。
ObjectDecoder | 基于 JDK 序列化来使用自定义序列化解码。外部依赖被排除在外时，提供了一个速度提升。否则选择其他序列化实现
ObjectEncoder | 基于 JDK 序列化来使用自定义序列化编码。外部依赖被排除在外时，提供了一个速度提升。否则选择其他序列化实现

### JBoss Marshalling 序列化

如果可以使用外部依赖 JBoss Marshalling 是个明智的选择。比 JDK 序列化快3倍且更加简练。

*[JBoss Marshalling](https://www.jboss.org/jbossmarshalling) 是另一个序列化 API,修复的许多 JDK序列化 API 中发现的问题,它与  java.io.Serializable 完全兼容。并添加了一些新的可调参数和附加功能,所有这些都可插入通过工厂配置外部化,类/实例查找表,类决议,对象替换,等等)*

下表展示了 Netty 支持 JBoss Marshalling 的编解码器。

Table 8.9 JBoss Marshalling codecs

名称 | 描述
-----|----
CompatibleMarshallingDecoder | 为了与使用 JDK 序列化的端对端间兼容。
CompatibleMarshallingEncoder | 为了与使用 JDK 序列化的端对端间兼容。
MarshallingDecoder | 使用自定义序列化用于解码，必须使用
MarshallingEncoder
MarshallingEncoder | 使用自定义序列化用于编码，必须使用
MarshallingDecoder

下面展示了使用 MarshallingDecoder 和 MarshallingEncoder

Listing 8.13 Using JBoss Marshalling
    
    public class MarshallingInitializer extends ChannelInitializer<Channel> {
    
        private final MarshallerProvider marshallerProvider;
        private final UnmarshallerProvider unmarshallerProvider;
    
        public MarshallingInitializer(UnmarshallerProvider unmarshallerProvider,
                                      MarshallerProvider marshallerProvider) {
            this.marshallerProvider = marshallerProvider;
            this.unmarshallerProvider = unmarshallerProvider;
        }
        @Override
        protected void initChannel(Channel channel) throws Exception {
            ChannelPipeline pipeline = channel.pipeline();
            pipeline.addLast(new MarshallingDecoder(unmarshallerProvider));
            pipeline.addLast(new MarshallingEncoder(marshallerProvider));
            pipeline.addLast(new ObjectHandler());
        }
    
        public static final class ObjectHandler extends SimpleChannelInboundHandler<Serializable> {
            @Override
            public void channelRead0(ChannelHandlerContext channelHandlerContext, Serializable serializable) throws Exception {
                // Do something
            }
        }
    }


### ProtoBuf 序列化

ProtoBuf 来自谷歌，并且开源了。它使编解码数据更加紧凑和高效。它已经绑定各种编程语言,使它适合跨语言项目。

下表展示了 Netty 支持 ProtoBuf 的 ChannelHandler 实现。

Table 8.10 ProtoBuf codec

名称 | 描述
-----|----
ProtobufDecoder | 使用 ProtoBuf 来解码消息
ProtobufEncoder | 使用 ProtoBuf 来编码消息 
ProtobufVarint32FrameDecoder | 在消息的整型长度域中，通过 "[Base 128 Varints](https://developers.google.com/protocol-buffers/docs/encoding)"将接收到的 ByteBuf 动态的分割

用法见下面

Listing 8.14 Using Google Protobuf

    public class ProtoBufInitializer extends ChannelInitializer<Channel> {
    
        private final MessageLite lite;
    
        public ProtoBufInitializer(MessageLite lite) {
            this.lite = lite;
        }
    
        @Override
        protected void initChannel(Channel ch) throws Exception {
            ChannelPipeline pipeline = ch.pipeline();
            pipeline.addLast(new ProtobufVarint32FrameDecoder());
            pipeline.addLast(new ProtobufEncoder());
            pipeline.addLast(new ProtobufDecoder(lite));
            pipeline.addLast(new ObjectHandler());
        }
    
        public static final class ObjectHandler extends SimpleChannelInboundHandler<Object> {
            @Override
            public void channelRead0(ChannelHandlerContext ctx, Object msg) throws Exception {
                // Do something with the object
            }
        }
    }


1. 添加 ProtobufVarint32FrameDecoder 用来分割帧
2. 添加 ProtobufEncoder 用来处理消息的编码
3. 添加 ProtobufDecoder 用来处理消息的解码
4. 添加 ObjectHandler 用来处理解码了的消息

本章在这最后一节中,我们探讨了 Netty 支持的不同的序列化的专门的解码器和编码器。这些是标准 JDK 序列化 API,JBoss Marshalling 和谷歌ProtoBuf。