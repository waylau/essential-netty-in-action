什么是 Bootstrapping 为什么要用
=========

Bootstrapping（引导） 是 Netty 中配置程序的过程，当你需要连接客户端或服务器绑定指定端口时需要使用 Bootstrapping。

如前面所述，Bootstrapping 有两种类型，一种是用于客户端的Bootstrap，一种是用于服务端的ServerBootstrap。不管程序使用哪种协议，无论是创建一个客户端还是服务器都需要使用“引导”。

*面向连接 vs. 无连接*

*请记住，这个讨论适用于 TCP 协议，它是“面向连接”的。这样协议保证该连接的端点之间的消息的有序输送。无连接协议发送的消息，无法保证顺序和成功性*

两种 Bootstrapping 之间有一些相似之处，也有一些不同。Bootstrap 和 ServerBootstrap 之间的差异如下：

Table 3.1 Comparison of Bootstrap classes

<table >
<tr>
  <td>分类</td>
  <td>Bootstrap</td>
  <td>ServerBootstrap</td>
</tr>
<tr>
  <td>网络功能</td>
  <td>连接到远程主机和端口</td>
  <td>绑定本地端口</td>
</tr>
<tr>
  <td>EventLoopGroup 数量</td>
  <td>1</td>
  <td>2</td>
</tr>
</table>

Bootstrap用来连接远程主机，有1个EventLoopGroup

ServerBootstrap用来绑定本地端口，有2个EventLoopGroup

事件组(Groups)，传输(transports)和处理程序(handlers)分别在本章后面讲述，我们在这里只讨论两种"引导"的差异(Bootstrap和ServerBootstrap)。第一个差异很明显，“ServerBootstrap”监听在服务器监听一个端口轮询客户端的“Bootstrap”或DatagramChannel是否连接服务器。通常需要调用“Bootstrap”类的connect()方法，但是也可以先调用bind()再调用connect()进行连接，之后使用的Channel包含在bind()返回的ChannelFuture中。

一个 ServerBootstrap 可以认为有2个 Channel 集合，第一个集合包含一个单例 ServerChannel，代表持有一个绑定了本地端口的 socket；第二集合包含所有创建的 Channel，处理服务器所接收到的客户端进来的连接。下图形象的描述了这种情况：

Figure 3.2 Server with two EventLoopGroups

![](../images/Figure 3.2 Server with two EventLoopGroups.jpg)

与 ServerChannel 相关 EventLoopGroup 分配一个 EventLoop 是
负责创建 Channels 用于传入的连接请求。一旦连接接受，第二个EventLoopGroup 分配一个 EventLoop 给它的 Channel。

