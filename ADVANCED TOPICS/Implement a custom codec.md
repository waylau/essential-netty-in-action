实现自定义的编解码器
====

本章介绍：

* Decoder
* Encoder
* 单元测试

本章讲述 Netty 中如何轻松实现定制的编解码器，由于 Netty 架构的灵活性，这些编解码器易于重用和测试。为了更容易实现，使用 Memcached 作为协议例子是因为它更方便我们实现。

Memcached 是来自 Memcached.org 的免费开源、高性能、分布式的内存对象缓存系统，其目的是加速动态 Web 应用程序的响应，减轻数据库负载；Memcache 实际上是一个以 key-value 存储任意数据的内存小块。可能有人会问“为什么使用 Memcached？”，因为 Memcached 协议非常简单，便于讲解。

