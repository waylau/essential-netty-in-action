Transport 使用情况
====

前面说了，并不是所有传输都支持核心协议，这会限制你的选择，具体看下表

Transport | TCP | UDP | SCTP*  |UDT
-------- | -------| ---- |---|---
NIO  | X  | X  | X  | X
OIO  | X  | X  | X  | X

*指目前仅在 Linux 上的支持。

*在 Linux 上启用 SCTP*

注意 SCTP 需要 kernel 支持，举例
Ubuntu：

	sudo apt-get install libsctp1

Fedora 使用 yum:

	sudo yum install kernel-modules-extra.x86_64 lksctp-tools.x86_64

虽然只有 [SCTP](http://www.ietf.org/rfc/rfc2960.txt) 具有这些特殊的要求，对应的特定的传输也有推荐的配置。想想也是，一个服务器平台可能会需要支持较高的数量的并发连接比单个客户端的话。

下面是你可能遇到的用例:

* OIO-在低连接数、需要低延迟时、阻塞时使用
* NIO-在高连接数时使用
* Local-在同一个JVM内通信时使用
* Embedded-测试ChannelHandler时使用

