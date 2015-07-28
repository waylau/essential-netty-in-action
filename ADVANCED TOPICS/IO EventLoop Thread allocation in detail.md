I/O EventLoop/Thread 分配细节
====

Netty 的使用一个包含 EventLoop 的 EventLoopGroup 为 Channel 的 I/O 和事件服务。EventLoop 创建并分配方式不同基于传输的实现。异步实现使用只有少数 EventLoop(和 Threads)共享于 Channel 之间
。这允许最小线程数服务多个 Channel,不需要为他们每个人都有一个专门的 Thread。

图15.7显示了如何使用 EventLoopGroup。

![](../images/Figure 15.7 Thread allocation for nonblocking transports (such as NIO and AIO).jpg)

1. 所有的 EventLoop 由 EventLoopGroup 分配。这里它将使用三个EventLoop 实例
2. 这个 EventLoop 处理所有分配给它管道的事件和任务。每个EventLoop 绑定到一个 Thread
3. 管道绑定到 EventLoop,所以所有操作总是被同一个线程在 Channel 的生命周期执行。一个管道属于一个连接

Figure 15.7 Thread allocation for nonblocking transports (such as NIO and AIO)


如图所述，使用有 3个 EventLoop （每个都有一个 Thread ） EventLoopGroup 。EventLoop （同时也是 Thread ）直接当 EventLoopGroup 创建时分配。这样保证资源是可以使用的

这三个 EventLoop 实例将会分配给每个新创建的 Channel。这是通过EventLoopGroup 实现,管理 EventLoop 实例。实际实现会照顾所有EventLoop 实例上均匀的创建 Channel (同样是不同的 Thread)。

一旦 Channel 是分配给一个 EventLoop,它将使用这个 EventLoop 其一生EventLoop 和同样的线程。你可以,也应该,依靠这个,因为它可以确保你不需要担心同步(包括线程安全、可见性和同步)在你ChannelHandler实现。
但是这也会影响使用ThreadLocal,例如,经常使用的应用程序。因为一个EventLoop通常权力多个频道,ThreadLocal将相同的频道分配给EventLoop。因此,它适合
状态跟踪等等。它仍然可以用于共享重或昂贵的对象之间的渠道,不再需要保持状态,因此它可以用于每个事件,而不需要依赖于先前ThreadLocal的状态。