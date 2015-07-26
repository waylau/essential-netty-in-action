SPDY
====

本章介绍

* SPDY 总览
* ChannelHandler, Decoder, 和 Encoder
* 引导一个基于 Netty 的应用
* 测试 SPDY/HTTPS

*[SPDY](http://www.chromium.org/spdy/spdy-whitepaper)(读作“speedy”)是一个谷歌开发的开放的网络协议，主要运用于 web 内容传输。SPDY 操纵 HTTP 流量,目标是减少 web 页面加载延迟,提高网络安全。SPDY 达到通过压缩、多路复用和优先级来减少延迟，虽然这取决于网络和网站部署条件的组合。“SPDY”这个名字是谷歌的一个商标,不是一个首字母缩写。（摘自<http://en.wikipedia.org/wiki/SPDY>）*

Netty 的包支持 SPDY。正如我们已经看到在其他情况下,这种支持将使您能够使用 SPDY 无需担心所有的内部细节。在这一章里,我们将提供你需要的所有信息关于在您的应用程序中启用 SPDY ，并同时支持 SPDY 和 HTTP。