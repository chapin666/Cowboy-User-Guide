# 现代 web

Cowboy是现代Web服务器。 本章解释了它的含义并详细说明了所涉及的所有标准。

Cowboy支持本文档中列出的所有标准。

## HTTP/2

HTTP/2是用于使用Web服务的最有效协议。 它使客户能够长时间保持连接打开; 同时发送请求; 通过HTTP头压缩减少请求的大小; 和更多。 该协议是二进制的，大大减少了解析它所需的资源。

HTTP/2还使服务器能够将消息推送到客户端。 这可以用于各种目的，包括在客户端请求之前发送相关资源，以努力减少延迟。 这也可用于启用双向通信。

Cowboy为HTTP/2提供透明支持。 了解它的客户可以使用它; 其他人自动回退到HTTP/1.1。

HTTP/2与HTTP/1.1语义兼容。

HTTP/2由RFC 7540和RFC 7541定义。

## HTTP/1.1

HTTP/1.1是HTTP协议的先前版本。 协议本身是基于文本的，并且存在许多问题和限制。 特别是不可能同时执行请求（尽管有时可以进行流水线操作），并且有时也很难检测到客户端断开连接。

HTTP/1.1确实为与Web服务交互提供了非常好的语义。 它定义了HTTP/1.1和HTTP/2客户端和服务器使用的标准方法，header和status。

HTTP/1.1还定义了与旧协议版本HTTP/1.0的兼容性，该协议从未在实现中实际标准化。

HTTP/1.1的核心由RFC 7230，RFC 7231，RFC 7232，RFC 7233，RFC 7234和RFC 7235定义。存在许多RFC和其他规范，用于定义其他HTTP方法，状态代码，header或语义。

## Websocket

[Websocket](https://ninenines.eu/docs/en/cowboy/2.4/guide/ws_protocol/)是一种建立在HTTP/1.1之上的协议，它在客户端和服务器之间提供双向通信通道。通信是异步的，可以同时发生。

它包含一个Javascript对象，允许建立与服务器的Websocket连接，以及一个基于二进制的协议，用于将数据发送到服务器或客户端。

Websocket连接可以传输UTF-8编码的文本数据或二进制数据。该协议还包括对实现ping / pong机制的支持，允许服务器和客户端更加确信连接仍然存在。

Websocket连接可用于传输任何类型的数据，无论是小型还是大型，文本还是二进制。因为这个Websocket有时用于系统之间的通信。

Websocket消息本身没有语义。 Websocket在这方面更接近TCP，并要求您在其上设计和实现自己的协议;或者将现有协议改编为Websocket。

Cowboy提供了一个称为[Websocket处理](https://ninenines.eu/docs/en/cowboy/2.4/guide/ws_handlers/)程序的接口，可以完全控制Websocket连接。

Websocket协议由RFC 6455定义。

## Long-lived requests

Cowboy提供了一个接口，可用于支持长轮询或可靠地流式传输大量数据，包括使用Server-Sent Events。

长轮询是客户端执行服务器可能无法立即应答的请求的机制。 它允许客户端请求当前可能不存在但预计很快就会创建的资源，并且会尽快返回。

长轮询本质上是一种hack，它被广泛用于克服旧客户端和服务器的限制。

Server-Sent Events是一种小型协议，定义为媒体类型，文本/事件流，以及新的HTTP header Last-Event-ID。 它在EventSource W3C规范中定义。

Cowboy提供了一个称为[循环处理](https://ninenines.eu/docs/en/cowboy/2.4/guide/loop_handlers/)程序的接口，它有助于实现长轮询或流机制。 无论底层协议如何，它都可以工作。

## REST

REST是[REpresentational State Transfer](https://ninenines.eu/docs/en/cowboy/2.4/guide/rest_principles/)的简写，它是松散连接分布式系统的一种架构风格。 它可以很容易地在HTTP之上实现。

REST本质上是一组要遵循的约束。 其中许多约束纯粹是架构的，只需使用HTTP即可解决。 开发人员必须明确遵循一些约束。

Cowboy提供了一个称为[REST处理](https://ninenines.eu/docs/en/cowboy/2.4/guide/rest_handlers/)程序的接口，它简化了在HTTP协议之上的REST API的实现。