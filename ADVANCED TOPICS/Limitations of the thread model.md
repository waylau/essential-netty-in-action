线程模型的局限
====

目前在 Netty 5 中仍然需要解决两个局限性,这些都存在于当前线程模型( Netty 4)。本节将解释这些局限性和显示团队计划如何在 Netty 5 中修复它们

### 选择下个 EventLoop

在 Netty 4 round-robin-like 算法用于给创建的 Channel 选择下一个 EventLoop。开始这工作得很好，但是一段时间后会发生一些 EventLoop 有些 Channel 分配比别人少。这是更有可能如果
EventLoopGroup(包含 EventLoop)处理不同的协议或不同类型的连接,如长连接和短连接。

修复可能听起来很容易,因为你可以选择 EventLoop 最小的 Channel 数。但考虑到渠道的数量 EventLoop 不是足够的适当的修复,因为即使一个EventLoop 持有多个 Channel。另一个它并不代表这 EventLoop 是最繁忙的。有很多需要考虑的事情。这些包括以下几点:

* 队列任务的数量
* 最后执行时间
* 饱和网络堆栈

问题已经在 <https://github.com/netty/netty/issues/1230> 进行了跟踪,目前针对网状的 5.0.0.Final 进行修复，但也可能向后移植到 Netty 4 。

### 整个 EventLoop 重新分配 Channel

另一个问题是如何分配 Channel 在 EventLoop,更好地利用资源。这对于长连接尤其重要,因为这样一些 EventLoop 会比别人忙。

为了修复,Netty 的团队目前正在引入 reregister(....) 方法,它允许您将一个 Channel 移动到另一个 EventLoop,因此对其处理转移到另一个Thread. 。但这不是像听起来那么容易,因为它带来了一些问题。

首先,您必须确保所有的事件和操作被触发，同时老 EventLoop 在老EventLoop 注册的同时执行。未能这样是因为这样将打破线程模型,不能保证每个 ChannelHandler 是由只有一个线程进行处理。另一个问题是如何确保一个线程移动到另一个你不遇到任何可见的问题。

这个问题目前正在跟踪作为 Netty 5.0.0.Final 的<47 https://github.com/netty/netty/issues/1799>。它不可能会移植到 Netty 4.x,因为它需要一个小的 API 破损暴露在 Channel 接口所需的操作上。





