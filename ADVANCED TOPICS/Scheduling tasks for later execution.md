调度任务执行
====

每隔一段时间需要调度任务执行，也许你想注册一个任务在客户端完成连接5分钟后执行，一个常见的用例是发送一个消息“你还活着？”到远端通，如果远端没有反应，则可以关闭通道(连接)和释放资源。
        
本节介绍使用强大的 EventLoop 实现任务调度，还会简单介绍 Java API的任务调度，以方便和 Netty 比较加深理解。

### 使用普通的 Java API 调度任务

在 Java 中使用 JDK 提供的 ScheduledExecutorService 实现任务调度。使用 Executors 提供的静态方法创建 ScheduledExecutorService，有如下方法

Table 15.1 java.util.concurrent.Executors-Static methods to create a ScheduledExecutorService

方法 | 描述
----|----
newScheduledThreadPool(int corePoolSize) newScheduledThreadPool(int corePoolSize,ThreadFactorythreadFactory) | | 创建一个新的
ScheduledThreadExecutorService 用于调度命令来延迟或者周期性的执行。 corePoolSize 用于计算线程的数量
newSingleThreadScheduledExecutor()  newSingleThreadScheduledExecutor(ThreadFact
orythreadFactory) | 新建一个 ScheduledThreadExecutorService 可以用于调度命令来延迟或者周期性的执行。它将使用一个线程来执行调度的任务

下面的 ScheduledExecutorService 调度任务 60 执行一次

Listing 15.4 Schedule task with a ScheduledExecutorService

    ScheduledExecutorService executor = Executors
            .newScheduledThreadPool(10); //1

    ScheduledFuture<?> future = executor.schedule(
            new Runnable() { //2
                @Override
                public void run() {
                    System.out.println("Now it is 60 seconds later");  //3
                }
            }, 60, TimeUnit.SECONDS);  //4
    // do something
    //

    executor.shutdown();  //5

1. 新建 ScheduledExecutorService 使用10个线程
2. 新建 runnable 调度执行
3. 稍后运行
4. 调度任务60秒后执行
5. 关闭 ScheduledExecutorService 来释放任务完成的资源

###  使用 EventLoop 调度任务

使用 ScheduledExecutorService 工作的很好，但是有局限性，比如在一个额外的线程中执行任务。如果需要执行很多任务，资源使用就会很严重；对于像 Netty 这样的高性能的网络框架来说，严重的资源使用是不能接受的。Netty 对这个问题提供了很好的方法。
        
Netty 允许使用 EventLoop 调度任务分配到通道，如下面代码：

Listing 15.5 Schedule task with EventLoop

    Channel ch = null; // Get reference to channel
    ScheduledFuture<?> future = ch.eventLoop().schedule(
            new Runnable() {
                @Override
                public void run() {
                    System.out.println("Now its 60 seconds later");
                }
            }, 60, TimeUnit.SECONDS);

1. 新建 runnable 用于执行调度
2. 稍后执行
3. 调度任务60秒后运行

如果想任务每隔多少秒执行一次，看下面代码：

Listing 15.6 Schedule a fixed task with the EventLoop

    Channel ch = null; // Get reference to channel
    ScheduledFuture<?> future = ch.eventLoop().scheduleAtFixedRate(
            new Runnable() {
                @Override
                public void run() {
                    System.out.println("Run every 60 seconds");
                }
            }, 60, 60, TimeUnit.SECONDS);

1. 新建 runnable 用于执行调度
2. 将运行直到  ScheduledFuture 被取消
3. 调度任务60秒运行

取消操作，可以使用 ScheduledFuture 返回每个异步操作。 ScheduledFuture 提供一个方法用于取消一个调度了的任务或者检查它的状态。一个简单的取消操作如下：

	ScheduledFuture<?> future = ch.eventLoop()
	.scheduleAtFixedRate(..); //1
	// Some other code that runs...
	future.cancel(false); //2

1. 调度任务并获取返回的 ScheduledFuture
2. 取消任务，阻止它再次运行

### 调度的内部实现

Netty 内部实现其实是基于George Varghese 提出的 “Hashed  and  hierarchical  timing wheels: Data structures  to efficiently implement timer facility(散列和分层定时轮：数据结构有效实现定时器)”。这种实现只保证一个近似执行，也就是说任务的执行可能不是100%准确；在实践中，这已经被证明是一个可容忍的限制，不影响多数应用程序。所以，定时执行任务不可能100%准确的按时执行。
        
为了更好的理解它是如何工作，我们可以这样认为：

* 在指定的延迟时间后调度任务；
* 任务被插入到 EventLoop 的 Schedule-Task-Queue(调度任务队列)；
* 如果任务需要马上执行，EventLoop 检查每个运行；
* 如果有一个任务要执行，EventLoop 将立刻执行它，并从队列中删除；
* EventLoop 等待下一次运行，从第4步开始一遍又一遍的重复。

因为这样的实现计划执行不可能100%正确，对于多数用例不可能100%准备的执行计划任务；在 Netty 中，这样的工作几乎没有资源开销。

但是如果需要更准确的执行呢？很容易，你需要使用ScheduledExecutorService 的另一个实现，这不是 Netty 的内容。记住，如果不遵循 Netty 的线程模型协议，你将需要自己同步并发访问。


