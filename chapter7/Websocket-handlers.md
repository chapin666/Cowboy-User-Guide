# Websocket 处理

Websocket处理程序提供了一个接口，用于升级到Websocket的HTTP/1.1连接以及在Websocket连接上发送或接收帧。

由于Websocket连接是通过HTTP/1.1升级机制建立的，因此Websocket处理程序需要能够在切换到Websocket并接管连接之前首先接收升级的HTTP请求。 然后，他们可以接收或发送Websocket帧，处理传入的Erlang消息或关闭连接。

## Upgrade

收到请求时，将调用init/2回调。 要建立Websocket连接，您必须切换到cowboy_websocket模块：
```
init(Req, State) ->
    {cowboy_websocket, Req, State}.
```

Cowboy会立即执行Websocket握手。 请注意，如果客户端未请求升级到Websocket，则握手将失败。

此函数返回后，Req对象变为不可用。 正确执行Websocket处理程序所需的任何信息都必须保存在该状态中。

## 子协议

客户端可以在sec-websocket-protocol header中提供它支持的Websocket子协议列表。 服务器必须选择其中一个并将其发送回客户端，否则握手将失败。

例如，客户端可以通过Websocket了解STOMP和MQTT，并提供header：

```
sec-websocket-protocol: v12.stomp, mqtt
```
如果服务器只理解MQTT，它可以返回：
```
sec-websocket-protocol: mqtt
```
此选择必须在init/2中完成。 示例用法如下：
```
init(Req0, State) ->
    case cowboy_req:parse_header(<<"sec-websocket-protocol">>, Req0) of
        undefined ->
            {cowboy_websocket, Req0, State};
        Subprotocols ->
            case lists:keymember(<<"mqtt">>, 1, Subprotocols) of
                true ->
                    Req = cowboy_req:set_resp_header(<<"sec-websocket-protocol">>,
                        <<"mqtt">>, Req0),
                    {cowboy_websocket, Req, State};
                false ->
                    Req = cowboy_req:reply(400, Req0),
                    {ok, Req, State}
            end
    end.
```

## 升级后初始化

Cowboy有单独的处理连接和请求的过程。 由于Websocket接管连接，因此Websocket协议处理发生在与请求处理不同的进程中。

这反映在Websocket处理程序具有的不同回调中。 从临时请求进程调用init/2回调，并从连接进程调用websocket_回调。

这意味着无法从init/2进行某些初始化。 任何需要当前pid或绑定到当前pid的内容都不会按预期工作。 可以使用可选的websocket_init/1代替：
```
websocket_init(State) ->
    erlang:start_timer(1000, self(), <<"Hello!">>),
    {ok, State}.
```
所有Websocket回调都共享相同的返回值。 这意味着我们可以在升级后立即向客户端发送帧：
```
websocket_init(State) ->
    {reply, {text, <<"Hello!">>}, State}.
```

## 接收帧

每当文本，二进制，ping或pong帧从客户端到达时，Cowboy都会调用websocket_handle/2。

处理程序可以处理或忽略帧。 它还可以将帧发送回客户端或停止连接。

以下代码段回显了所有收到的文本帧并忽略了所有其他帧：
```
websocket_handle(Frame = {text, _}, State) ->
    {reply, Frame, State};
websocket_handle(_Frame, State) ->
    {ok, State}.
```

## 接收Erlang消息

每当Erlang消息到达时，Cowboy都会调用websocket_info 2。

处理程序可以处理或忽略消息。 它还可以将帧发送到客户端或停止连接。

以下代码段将日志消息转发给客户端并忽略所有其他消息：
```
websocket_info({log, Text}, State) ->
    {reply, {text, Text}, State};
websocket_info(_Info, State) ->
    {ok, State}.
```

## 发送帧

所有websocket_回调都共享返回值。 他们可以向客户端发送零个，一个或多个帧。

不发送任何内容，只需返回一个ok元组：
```
websocket_info(_Info, State) ->
    {ok, State}.
```

要发送一个帧，请返回一个带有要发送的帧的回复元组：
```
websocket_info(_Info, State) ->
    {reply, {text, <<"Hello!">>}, State}.
```

您可以发送任何类型的帧：文本，二进制，ping，pong或关闭帧。

要一次发送多个帧，请返回一个回复元组，其中包含要发送的帧列表：
```
websocket_info(_Info, State) ->
    {reply, [
        {text, "Hello"},
        {text, <<"world!">>},
        {binary, <<0:8000>>}
    ], State}.
```
它们按给定顺序发送。

## 保持连接活着

Cowboy会自动响应客户端发送的ping帧。 它们仍然被转发给处理程序以用于提供信息，但不需要进一步的操作。

Cowboy本身不发送ping帧。 处理程序可以根据需要执行此操作。 在大多数情况下，更好的解决方案是让客户端处理ping。 从处理程序执行此操作意味着每个连接都有一个额外的计时器，这对于需要处理大量连接的服务器来说可能是相当大的成本。

Cowboy可以配置为自动关闭空闲连接。 强烈建议在此处配置超时，以避免进程延迟超过需要的时间。

init/2回调可以设置用于连接的超时。 例如，这会使Cowboy关闭连接空闲超过30秒：
```
init(Req, State) ->
    {cowboy_websocket, Req, State, #{
        idle_timeout => 30000}}.
```
设置后，无法更改此值。 默认为60000

## 限制帧大小

Cowboy默认接受任何大小的帧。 您应该根据处理程序可能处理的内容来限制大小。 您可以通过init/2回调执行此操作：
```
init(Req, State) ->
    {cowboy_websocket, Req, State, #{
        max_frame_size => 8000000}}.
```
缺乏限制是历史性的。 Cowboy的未来版本将有更合理的默认值。

## 节省内存

在回调返回后，可以将Websocket连接进程设置为休眠。

只需将一个休眠字段添加到ok或回复元组：

```
websocket_init(State) ->
    {ok, State, hibernate}.

websocket_handle(_Frame, State) ->
    {ok, State, hibernate}.

websocket_info(_Info, State) ->
    {reply, {text, <<"Hello!">>}, State, hibernate}.
```
强烈建议您在启用了休眠的情况下编写处理程序，因为这样可以大大减少内存使用量。 但请注意，可以观察到CPU使用率或延迟的增加，特别是对于更繁忙的连接。

## 关闭连接

可以随时关闭连接，通过告诉Cowboy停止连接或发送关闭帧。

要告诉Cowboy关闭连接，请使用stop tuple:
```
websocket_info(_Info, State) ->
    {stop, State}.
```

发送关闭帧将立即启动Websocket连接的关闭。 请注意，在发送包含关闭帧的帧列表时，不会发送在关闭帧之后找到的任何帧。

以下示例发送带有原因消息的关闭帧：
```
websocket_info(_Info, State) ->
    {reply, {close, 1000, <<"some-reason">>}, State}.
```