# Streams

Stream 是形成HTTP请求/响应对的消息集。

Stream 来自HTTP/2。 在Cowboy中，它也在HTTP/1.1或HTTP/1.0时谈论使用。 它不应与流式传输请求或响应主体混淆。

所有版本的HTTP都允许客户端启动stream。 HTTP/2是唯一一个也允许服务器通过其服务器推送功能。 客户端和服务器启动的流都在Cowboy中执行相同的过程。

## Stream 处理

流处理程序必须实现五种不同的回调。其中四个是直接相关的;一个是特别的。

所有回调都接收流ID作为第一个参数。

他们中的大多数都可以返回Cowboy执行的命令列表。当链接回调时，可以拦截和修改这些命令。例如，这对于修改响应非常有用。

当有新请求进入时，将调用init/3回调。它接收Req对象和此侦听器的协议选项。

当收到来自请求主体的数据时，将调用data/4回调。它接收这个数据和一个标志，表明是否有更多的预期。

收到此流的Erlang消息时，将调用info/3回调。它们通常是请求过程发送的消息。

最后，使用流的终止原因调用terminate/3回调。返回值被忽略。请注意，与Erlang中的所有终止回调一样，没有强有力的保证它将被调用。

在完全接收到请求标头并且Cowboy正在发送响应之前发生错误时，将调用特殊回调early_error/5。它接收部分Req对象，错误原因，协议选项和Cowboy将发送的响应。必须返回此响应，可能已修改。

## 内置处理程序

Cowboy有两个处理程序。

cowboy_stream_h是默认的流处理程序。 它是Cowboy大部分功能的核心。 流处理程序的所有链都应该最后调用它。

cowboy_compress_h会在可能的情况下自动压缩响应。 默认情况下不启用它。 这是编写自己的处理程序的一个很好的例子，它将修改响应。