单元测试
====

本章介绍

* 单元测试
* EmbeddedChannel
       
学会了使用一个或多个 ChannelHandler 处理接收/发送数据消息，但是如何测试它们呢？Netty 提供了2个额外的类使得测试 ChannelHandler变得很容易，本章讲解如何测试 Netty 程序。测试使用 JUnit4，如果不会用可以慢慢了解。JUnit4 很简单，但是功能很强大。

本章将重点讲解测试已实现的 ChannelHandler 和编解码器