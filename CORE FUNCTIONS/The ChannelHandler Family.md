ChannelHandler 家族
====

在我们深入研究 ChannelHandler 内部之前，让我们花几分钟了解下这个 Netty 组件模型的基础。这里先对ChannelHandler 及其子类做个简单的介绍。

### Channel 生命周期

Channel 有个简单但强大的状态模型，与
ChannelInboundHandler API  密切相关。下面表格是 Channel 的四个状态

Table 6.1 Channel lifeycle states

状态 | 描述
-----|---------
channelUnregistered  | channel已创建但未注册到一个 EventLoop.
channelRegistered  | channel 注册到一个 EventLoop.
channelActive | channel 变为活跃状态(连接到了远程主机)，现在可以接收和发送数据了
channelInactive  | channel 处于非活跃状态，没有连接到远程主机

Channel 的正常的生命周期如下图，当状态出现变化，就会触发对应的事件，这样就能与 ChannelPipeline 中的 ChannelHandler进行及时的交互。 

Figure 6.1 Channel State Model

![](../images/Figure 6.1 Channel State Model.jpg)

###ChannelHandler 生命周期

ChannelHandler 定义的生命周期操作如下表，当 ChannelHandler 添加到 ChannelPipeline，或者从 ChannelPipeline 移除后，对应的方法将会被调用。每个方法都传入了一个 ChannelHandlerContext 参数

Table 6.2 ChannelHandler lifecycle methods

类型 | 描述
-----|---------
handlerAdded  | 当 ChannelHandler 添加到 ChannelPipeline 调用
handlerRemoved | 当 ChannelHandler 从 ChannelPipeline 移除时调用
exceptionCaught | 当 ChannelPipeline 执行抛出异常时调用

###ChannelHandler 子接口

Netty 提供2个重要的 ChannelHandler 子接口：

* ChannelInboundHandler - 处理进站数据和所有状态更改事件
* ChannelOutboundHandler - 处理出站数据，允许拦截各种操作

*ChannelHandler 适配器*

*Netty 提供了一个简单的 ChannelHandler 框架实现，给所有声明方法签名。这个类 ChannelHandlerAdapter 的方法,主要推送事件 到 pipeline 下个 ChannelHandler 直到 pipeline 的结束。这个类
也作为 ChannelInboundHandlerAdapter 和ChannelOutboundHandlerAdapter 的基础。所有三个适配器类的目的是作为自己的实现的起点;您可以扩展它们,覆盖你需要自定义的方法。*

###ChannelInboundHandler 

ChannelInboundHandler  的生命周期方法在下表中，当接收到数据或者与之关联的 Channel 状态改变时调用。之前已经注意到了，这些方法与 Channel 的生命周期接近

Table 6.3 ChannelInboundHandler methods

类型 | 描述
-----|---------
channelRegistered  | Invoked when a Channel is registered to its EventLoop and is able to handle I/O.
channelUnregistered  | Invoked when a Channel is deregistered from its EventLoop and cannot handle any I/O.
channelActive  | Invoked when a Channel is active; the Channel is connected/bound and ready.
channelInactive | Invoked when a Channel leaves active state and is no longer connected to its remote peer.
channelReadComplete  |  Invoked when a read operation on the Channel has completed.
channelRead  | Invoked if data are read from the Channel.
channelWritabilityChanged |  Invoked when the writability state of the Channel changes. The user can ensure writes are not done too fast (with risk of an OutOfMemoryError) or can resume writes when the Channel becomes writable again.Channel.isWritable() can be used to detect the actual writability of the channel. The threshold for writability can be set via Channel.config().setWriteHighWaterMark() and Channel.config().setWriteLowWaterMark().
userEventTriggered(...) | Invoked when a user calls Channel.fireUserEventTriggered(...) to pass a pojo through the ChannelPipeline. This can be used to pass user specific events through the ChannelPipeline and so allow handling those events.

注意，ChannelInboundHandler 实现覆盖了 channelRead() 方法处理进来的数据用来响应释放资源。Netty 在 ByteBuf 上使用了资源池，所以当执行释放资源时可以减少内存的消耗。

Listing 6.1 Handler to discard data
 
	@ChannelHandler.Sharable
	public class DiscardHandler extends ChannelInboundHandlerAdapter {		//1
	
	    @Override
	    public void channelRead(ChannelHandlerContext ctx,
	                                     Object msg) {
	        ReferenceCountUtil.release(msg); //2
	    }
	
	}

1.扩展 ChannelInboundHandlerAdapter

2.ReferenceCountUtil.release() 来丢弃收到的信息

Netty 用一个 WARN-level 日志条目记录未释放的资源,使其能相当简单地找到代码中的违规实例。然而,由于手工管理资源会很繁琐,您可以通过使用 SimpleChannelInboundHandler 简化问题。如下：

Listing 6.2 Handler to discard data

	@ChannelHandler.Sharable
	public class SimpleDiscardHandler extends SimpleChannelInboundHandler<Object> {  //1
	
	    @Override
	    public void channelRead0(ChannelHandlerContext ctx,
	                                     Object msg) {
			// No need to do anything special //2
	    }
	
	}

1.扩展 SimpleChannelInboundHandler

2.不需做特别的释放资源的动作

注意 SimpleChannelInboundHandler 会自动释放资源，而无需存储任何信息的引用。

更多详见 “Error! Reference source not found..” 一节

###ChannelOutboundHandler 

ChannelOutboundHandler 提供了出站操作时调用的方法。这些方法会被 Channel, ChannelPipeline, 和 ChannelHandlerContext 调用。

ChannelOutboundHandler 另个一个强大的方面是它具有在请求时延迟操作或者事件的能力。比如，当你在写数据到 remote peer 的过程中被意外暂停，你可以延迟执行刷新操作，然后在迟些时候继续。

下面显示了 ChannelOutboundHandler 的方法（继承自 ChannelHandler 未列出来）


Table 6.4 ChannelOutboundHandler methods

类型 | 描述
-----|---------
bind  | Invoked on request to bind the Channel to a local address
connect |  Invoked on request to connect the Channel to the remote peer
disconnect |  Invoked on request to disconnect the Channel from the remote peer
close  | Invoked on request to close the Channel
deregister |  Invoked on request to deregister the Channel from its EventLoop
read  | Invoked on request to read more data from the Channel
flush  | Invoked on request to flush queued data to the remote peer through the Channel
write  | Invoked on request to write data through the Channel to the remote peer

几乎所有的方法都将 ChannelPromise 作为参数,一旦请求结束要通过 ChannelPipeline 转发的时候，必须通知此参数。

*ChannelPromise vs. ChannelFuture*

*ChannelPromise 是 特殊的 ChannelFuture，允许你的 ChannelPromise 及其 操作 成功或失败。所以任何时候调用例如 Channel.write(...) 一个新的  ChannelPromise将会创建并且通过 ChannelPipeline传递。这次写操作本身将会返回 ChannelFuture， 这样只允许你得到一次操作完成的通知。Netty 本身使用 ChannelPromise 作为返回的 ChannelFuture 的通知，事实上在大多数时候就是 ChannelPromise 自身（ChannelPromise 扩展了 ChannelFuture）*

如前所述,ChannelOutboundHandlerAdapter 提供了一个实现了 ChannelOutboundHandler 所有基本方法的实现的框架。
这些简单事件转发到下一个 ChannelOutboundHandler 管道通过调用
ChannelHandlerContext 相关的等效方法。你可以根据需要自己实现想要的方法。

### 资源管理

当你通过 ChannelInboundHandler.channelRead(...) 或者
ChannelOutboundHandler.write(...) 来处理数据，重要的是在处理资源时要确保资源不要泄漏。

Netty 使用引用计数器来处理池化的 ByteBuf。所以当 ByteBuf 完全处理后，要确保引用计数器被调整。

引用计数的权衡之一是用户时必须小心使用消息。当 JVM 仍在 GC(不知道有这样的消息引用计数)这个消息，以至于可能是之前获得的这个消息不会被放回池中。因此很可能,如果你不小心释放这些消息，很可能会耗尽资源。

为了让用户更加简单的找到遗漏的释放，Netty 包含了一个 ResourceLeakDetector ，将会从已分配的缓冲区 1% 作为样品来检查是否存在在应用程序泄漏。因为 1% 的抽样,开销很小。

对于检测泄漏,您将看到类似于下面的日志消息。

	LEAK: ByteBuf.release() was not called before it’s garbage-collected. Enable advanced leak reporting to find out where the leak occurred. To enable advanced
	leak reporting, specify the JVM option ’-Dio.netty.leakDetectionLevel=advanced’ or call ResourceLeakDetector.setLevel()

	Relaunch your application with the JVM option mentioned above, then you’ll see the recent locations of your application where the leaked buffer was accessed. The following output shows a leak from our unit test (XmlFrameDecoderTest.testDecodeWithXml()):
		
	Running io.netty.handler.codec.xml.XmlFrameDecoderTest
		
	15:03:36.886 [main] ERROR io.netty.util.ResourceLeakDetector - LEAK:
	ByteBuf.release() was not called before it’s garbage-collected.
		
	Recent access records: 1
	
	#1:
		
	io.netty.buffer.AdvancedLeakAwareByteBuf.toString(AdvancedLeakAwareByteBuf.java:697)
	
	io.netty.handler.codec.xml.XmlFrameDecoderTest.testDecodeWithXml(XmlFrameDecoderTest.java:157)
		io.netty.handler.codec.xml.XmlFrameDecoderTest.testDecodeWithTwoMessages(XmlFrameDecoderTest.java:133)


#### 泄漏检测等级

Netty 现在定义了四种泄漏检测等级，可以按需开启，见下表


Table 6.5 Leak detection levels

Level Description | DISABLED
------------------|---------
Disables | Leak detection completely. While this even eliminates the 1 % overhead you should only do this after extensive testing.
SIMPLE | Tells if a leak was found or not. Again uses the sampling rate of 1%, the default level and a good fit for most cases.
ADVANCED | Tells if a leak was found and where the message was accessed, using the sampling rate of 1%.
PARANOID | Same as level ADVANCED with the main difference that every access is sampled. This it has a massive impact on performance. Use this only in the debugging phase.

修改检测等级，只需修改 io.netty.leakDetectionLevel 系统属性，举例

	# java -Dio.netty.leakDetectionLevel=paranoid

这样，我们就能在 ChannelInboundHandler.channelRead(...) 和  ChannelOutboundHandler.write(...) 避免泄漏。

当你处理 channelRead(...) 操作，并在消费消息(不是通过 ChannelHandlerContext.fireChannelRead(...) 来传递它到下个 ChannelInboundHandler)  时，要释放它，如下：

Listing 6.3 Handler that consume inbound data

	@ChannelHandler.Sharable
	public class DiscardInboundHandler extends ChannelInboundHandlerAdapter {  //1
	
	    @Override
	    public void channelRead(ChannelHandlerContext ctx,
	                                     Object msg) {
	        ReferenceCountUtil.release(msg); //2
	    }
	
	}

1. 继承 ChannelInboundHandlerAdapter
2. 使用 ReferenceCountUtil.release(...) 来释放资源

所以记得，每次处理消息时，都要释放它。

*SimpleChannelInboundHandler -消费入站消息更容易*

*使用入站数据和释放它是一项常见的任务，Netty 为你提供了一个特殊的称为 SimpleChannelInboundHandler 的 ChannelInboundHandler 的实现。该实现将自动释放一个消息，一旦这个消息被用户通过channelRead0() 方法消费。*

当你在处理写操作，并丢弃消息时，你需要释放它。现在让我们看下实际是如何操作的。

Listing 6.4 Handler to discard outbound data
 
@ChannelHandler.Sharable
public class DiscardOutboundHandler
        extends ChannelOutboundHandlerAdapter { //1

    @Override
    public void write(ChannelHandlerContext ctx,
                                     Object msg, ChannelPromise promise) {
        ReferenceCountUtil.release(msg);  //2
        promise.setSuccess();	//3

    }

}


1. 继承 ChannelOutboundHandlerAdapter
2. 使用 ReferenceCountUtil.release(...) 来释放资源
3. 通知 ChannelPromise 数据已经被处理

重要的是，释放资源并通知 ChannelPromise。如果，ChannelPromise 没有被通知到，这可能会引发 ChannelFutureListener 不会被处理的消息通知的状况。

所以，总结下：如果消息是被 消耗/丢弃 并不会被传入下个 ChannelPipeline 的 ChannelOutboundHandler ，调用 ReferenceCountUtil.release(message) 。一旦消息经过实际的传输，在消息被写或者 Channel 关闭时，它将会自动释放。
