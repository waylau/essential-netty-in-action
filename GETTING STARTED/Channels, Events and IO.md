Channel, Event 和 I/O
====

Netty 是一个非阻塞、事件驱动的网络框架。Netty 实际上是使用 Threads（多线程）处理 I/O 事件，对于熟悉多线程编程的读者可能会需要关注同步代码。这样的方式不好，因为同步会影响程序的性能，Netty 的设计保证程序处理事件不会有同步。图 Figure 3.1 展示了，你不需要在 Channel 之间共享 ChannelHandler 实例的原因：

Figure 3.1 

![](../images/Figure 3.1.jpg)

该图显示，一个 EventLoopGroup 具有一个或多个 EventLoop。想象 EventLoop 作为一个线程执行一个通道的 Channel 工作。 （事实上，一个 EventLoop 是势必为它的生命周期一个线程。）
当创建一个通道，Netty的注册该通道的单一事件循环实例
（并因此到单个螺纹）的通道的使用寿命。这就是为什么你的应用程序
没有按吨需要对Netty的I / O操作同步？;所有的I/ O对于给定的通道将
始终用相同的线程来执行。
我们将在第15章进一步讨论事件循环和EventLoopGroup。