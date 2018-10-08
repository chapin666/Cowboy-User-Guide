# Websocket 协议

本章解释了Websocket是什么以及为什么它是软实时Web应用程序的重要组成部分。

## 描述

Websocket是HTTP的扩展，它模拟客户端（通常是Web浏览器）和服务器之间的纯TCP连接。 它使用HTTP Upgrade机制建立连接。

Websocket连接完全异步，与HTTP/1.1（同步）和HTTP/2（异步，但服务器只能响应请求启动流）不同。 使用Websocket，客户端和服务器都可以随时发送帧而不受任何限制。 它比任何HTTP协议都更接近TCP。

Websocket是IETF标准。 Cowboy支持以前由浏览器实现的标准和所有草稿，不包括有时被称为“版本0”的初始缺陷草案。

## Websocket vs HTTP/2

几年来，Websocket是与服务器进行双向异步连接的唯一方法。 这在引入HTTP/2时发生了变化。 虽然HTTP/2要求客户端在服务器可以推送数据之前首先执行请求，但这只是一个小的限制因为客户端可以像连接一样这样做。

Websocket被设计为服务器的一种TCP通道。 它只定义了框架和连接管理，并允许开发人员在其上实现协议。 例如，您可以通过Websocket实现IRC并使用Javascript IRC客户端与服务器通信。

另一方面，HTTP/2只是对HTTP/1.1连接和请求/响应机制的改进。 它具有与HTTP/1.1相同的语义。

如果您只需要访问HTTP API，那么HTTP/2应该是您的首选。 另一方面，如果您需要的是不同的协议，那么您可以使用Websocket来实现它。

## 实现

Cowboy将Websocket实现为协议升级。 一旦从init/2回调执行升级，Cowboy就会切换到Websocket。 有关启动和处理Websocket连接的更多信息，请参阅下一章。

使用Autobahn测试套件验证Cowboy中Websocket的实现，该测试套件是一套涵盖协议所有方面的广泛测试。 Cowboy通过该套件100％成功，包括所有可选测试。

Cowboy的Websocket实现还包括permessage-deflate和x-webkit-deflate-frame压缩扩展。

从init/2函数返回compress选项时，Cowboy将自动使用压缩。