SPDY 背景
====

Google 开发 SPDY 是为了解决扩展性的问题。主要的任务是加载内容的速度更快，做了如下工作：

* 每个头都是压缩的，消息体的压缩是可选的,因为它可能对代理服务器有问题
* 所有的加密都使用 [TLS](http://en.wikipedia.org/wiki/Transport_Layer_Security)
*每个连接多个转移是可能的
*数据集可以单独设置优先级,使关键内容先被转移

下表是与 HTTP 的对比

Table 12.1 Comparison of SPDY and HTTP

浏览器 | HTTP 1.1 | SPDY
--------|----------|------
加密 | Not by default | Yes
Header 压缩 | No | Yes
全双工 | No | Yes
Server push | No | Yes
优先级 | No | Yes

一些使用场合和指标显示，可以 SPDY 让页面加载速度比H TTP 原先快50%。

现在 SPDY 的协议草案规范是 1, 2 和 3， Netty 支持 2和3，主要考虑到这个是被广大浏览器所支持的版本。
现在很多浏览器都支持 SPDY，见下表：

Table 12.2 Browsers that support SPDY

浏览器 | 版本
------|------
Chrome | 19+
Chromium | 19+
Mozilla Firefox | 11+ (从 13 起默认开启)
Opera | 12.10+

