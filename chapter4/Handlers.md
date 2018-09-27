# Handlers

Handlers是处理HTTP请求的Erlang模块。

## 普通 HTTP 处理

Cowboy中最基本的处理程序实现强制init/2回调，操作请求，可选地发送响应然后返回。

此回调接收Req对象和路由器配置中定义的初始状态。

不执行任何操作的处理程序将如下所示：
```
init(Req, State) ->
    {ok, Req, State}.
```

尽管没有发送回复，但仍会向客户端发送204 No Content响应，因为Cowboy会确保为每个请求发送响应。

我们需要使用Req对象来回复。

```
init(Req0, State) ->
    Req = cowboy_req:reply(200, #{
        <<"content-type">> => <<"text/plain">>
    }, <<"Hello World!">>, Req0),
    {ok, Req, State}.
```

当Cowboy：reply/4被调用时，Cowboy会立即发送回复。

然后我们返回一个3元组。 ok表示处理程序成功运行。 我们还将修改后的Req送回Cowboy。

元组的最后一个值是将在此处理程序的每个后续回调中使用的状态。 普通HTTP处理程序只有一个额外的回调，可选且很少使用terminate/3。

## 其他处理

init/2回调也可用于告知Cowboy这是一种不同类型的处理程序，Cowboy应切换到它。 要执行此操作，您只需返回要切换到的处理程序类型的模块名称。

Cowboy带有三种处理器类型，你可以切换到：cowboy_rest，cowboy_websocket和cowboy_loop。 除了那些，您还可以定义自己的处理程序类型。

切换很简单。 您只需返回要使用的处理程序类型的名称，而不是返回ok。 以下代码段切换到Websocket处理程序：
```
init(Req, State) ->
    {cowboy_websocket, Req, State}.
```

## Cleaning up

所有处理程序类型都提供可选的terminate/3回调。
```
terminate(_Reason, _Req, _State) ->
    ok.
```

此回调严格保留用于任何所需的清理。 您无法从此功能发送回复。 没有其他返回值。

此回调是可选的，因为它很少需要。 清理应该直接在单独的进程中完成（通过监视处理程序进程来检测它何时退出）。

Cowboy不会为不同的请求重用流程。 此调用返回后，该过程将很快终止。