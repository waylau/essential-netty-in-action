实现 Memcached 编解码器
====


当想要实现一个给定协议的编解码器，我们应该花一些事件来了解它的运作原理。通常情况下，协议本身都有一些详细的记录。在这里你会发现多少细节？幸运的是 Memcached 的二进制协议可以很好的扩展。     
在 RFC 中有相应的规范，可以在 <https://code.google.com/p/Memcached/wiki/MemcacheBinaryProtocol> 找到 。

我们不会实现 Memcached 的所有命令，只会实现三种操作：SET,GET 和 DELETE。这样做事为了让事情变得简单。

