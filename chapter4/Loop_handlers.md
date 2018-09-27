# Loop handlers

循环处理程序是一种特殊的HTTP处理程序，当响应无法立即发送时使用。处理程序在发送响应之前输入等待正确消息的接收循环。

循环处理程序用于响应可能无法立即可用的请求，但您希望在响应到达时保持连接打开一段时间。这种做法最着名的例子被称为长轮询。

循环处理程序也可用于响应部分可用的请求，并且您需要在连接打开时流式传输响应正文。这种做法最着名的例子是服务器发送的事件。

虽然使用普通HTTP处理程序可以实现相同的功能，但建议使用循环处理程序，因为它们经过了充分测试，并允许使用内置功能，如休眠和超时。

循环处理程序基本上等待一条或多条Erlang消息，并将这些消息提供给info/3回调。它还具有init/2和terminate/3回调，它们与普通HTTP处理程序的工作方式相同。

## 初始化

init/2函数必须返回cowboy_loop元组以启用循环处理程序行为。 该元组可以可选地包含超时值和/或原子休眠，以使进程进入休眠状态直到收到消息。

此代码段启用循环处理程序：
```
init(Req, State) ->
    {cowboy_loop, Req, State}.
```
也可以使进程休眠：
```
init(Req, State) ->
    {cowboy_loop, Req, State, hibernate}.
```

## Receive loop

初始化后，Cowboy将等待消息到达进程邮箱。 当消息到达时，Cowboy使用message，Req对象和处理程序的状态调用info/3函数。

以下代码段在收到来自其他进程的回复消息时发送回复，否则等待另一条消息。
```
info({reply, Body}, Req, State) ->
    cowboy_req:reply(200, #{}, Body, Req),
    {stop, Req, State};
info(_Msg, Req, State) ->
    {ok, Req, State, hibernate}.
```

请注意，此处的回复元组可能是任何消息，只是一个示例。

该回调可以执行任何必要的操作，包括发送全部或部分回复，并且随后将返回指示是否预期有更多消息的元组。

回调也可以选择什么也不做，只是跳过收到的消息。

如果发送了回复，则应返回停止元组。 这将指示Cowboy结束请求。

否则应该返回一个ok元组。

## Streaming loop

另一个非常适合循环处理程序的常见情况是以Erlang消息的形式接收的流数据。 这可以通过在init/2回调中启动分块回复，然后在每次收到消息时使用cowboy_req:chunk/2来完成。

以下代码片段就是这样做的。 正如您所看到的，每次收到事件消息时都会发送一个块，并通过发送eof消息来停止循环。
```
init(Req, State) ->
    Req2 = cowboy_req:stream_reply(200, Req),
    {cowboy_loop, Req2, State}.

info(eof, Req, State) ->
    {stop, Req, State};
info({event, Data}, Req, State) ->
    cowboy_req:stream_body(Data, nofin, Req),
    {ok, Req, State};
info(_Msg, Req, State) ->
    {ok, Req, State}.
```

## Cleaning up

建议您在回复时将连接Header设置为关闭，因为此过程可能会重复用于后续请求。

有关清理的一般说明，请参阅“[处理程序](https://ninenines.eu/docs/en/cowboy/2.4/guide/handlers/)”一章。

## 休眠

为了节省内存，您可以在收到的消息之间休眠该过程。 这是通过返回原子休眠作为循环元组回调通常返回的一部分来完成的。 只需在末尾添加原子，Cowboy就会相应地休眠。