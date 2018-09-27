# 读取 request 内容

可以使用Req对象读取请求正文。

Cowboy在被要求之前不会尝试阅读body。 您需要调用body reading函数才能检索它。

Cowboy不会缓存body，因此只能读一次。

但是，您无需阅读。 如果body存在且未被阅读，Cowboy将取消或跳过其下载，具体取决于协议。

Cowboy提供阅读body原始的功能，并阅读和解析形式urlencoded或多部分的身体。 后者在其章节中有所涉及。

## 请求body存在

并非所有请求都带有body。 您可以使用此功能检查是否存在请求body：
```
cowboy_req:has_body(REQ).
```
如果有body，它返回true; 否则是假的。

## body 长度

你可以获得身体的长度：

length = cowboy_req:body_length(Req)。
请注意，可能事先不知道长度。 在这种情况下，将返回undefined。 这可能发生在HTTP/1.1的分块传输编码或HTTP/2时，如果没有提供内容长度。

一旦完全读取了body，Cowboy将更新Req对象的body。 尝试在完全读取body后调用此函数时，将始终返回长度。

## 读取 body

您可以通过一个函数调用读取整个body：
```
{ok, Data, Req} = cowboy_req:read_body(Req0).
```

当body完全被读取时，Cowboy会返回一个好的元组。

默认情况下，Cowboy将尝试读取最多8MB的数据，最长可达15秒。 一旦读取了至少8MB的数据，或者在15秒结束时，该调用将返回。

这些值可以自定义。 例如，最多只能读取1MB，最多5秒：
```
{ok, Data, Req} = cowboy_req:read_body(Req0,
    #{length => 1000000, period => 5000}).
```
您也可以禁用长度限制：
```
{ok, Data, Req} = cowboy_req:read_body(Req0, #{length => infinity}).
```

这使得函数等待15秒并返回该期间到达的任何内容。 不建议面向公众的应用程序使用此功能。

这两个选项可以有效地用于控制请求体的传输速率。

### body stream

当body太大时，第一次调用将返回更多元组而不是ok。 你可以再次调用该函数来读取更多的body，一次读取一个块。
```
read_body_to_console(Req0) ->
    case cowboy_req:read_body(Req0) of
        {ok, Data, Req} ->
            io:format("~s", [Data]),
            Req;
        {more, Data, Req} ->
            io:format("~s", [Data]),
            read_body_to_console(Req)
    end.
```

也可以使用长度和周期选项。 他们需要为每次调用都通过。


## 从urlencoded body读取

Cowboy提供了一个方便的功能，用于读取和解析作为application/x-www-form-urlencoded发送的body。
```
{ok, KeyValues, Req} = cowboy_req:read_urlencoded_body(Req0).
```

此函数返回键/值列表，与函数cowboy_req:parse_qs/1完全相同。

此功能的默认值不同。 Cowboy最长可读取64KB，最长可达5秒。 它们可以修改：
```
{ok, KeyValues, Req} = cowboy_req:read_urlencoded_body(Req0,
    #{length => 4096, period => 3000}).
```