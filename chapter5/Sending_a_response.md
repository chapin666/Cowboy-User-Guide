# 发送 response

必须使用Req对象发送响应。

Cowboy提供了两种不同的发送响应方式：直接或通过流体传输。 响应标题和正文可以提前设置。 一旦调用了回复或流回复函数之一，就会发送响应。

Cowboy还提供了一个简化的发送文件界面。 它也可以只发送文件的特定部分。

虽然每个请求只允许一个响应，但HTTP/2引入了一种机制，允许服务器推送与响应相关的其他资源。 本章还介绍了此功能在Cowboy中的工作原理。

## Reply

Cowboy 提供三种函数来发送整个回复，具体取决于您是否需要设置标题和正文。 在所有情况下，Cowboy都会添加协议所需的任何header（例如，始终会发送日期header）。

当您只需要设置状态代码时，请使用cowboy_req:reply/2：
```
Req = cowboy_req:reply(200, Req0).
```
当您需要同时设置响应header时，请使用cowboy_req:reply/3：
```
Req = cowboy_req:reply(303, #{
    <<"location">> => <<"https://ninenines.eu">>
}, Req0).
```

请注意，header名称必须始终为小写二进制文件。

当您还需要设置响应体时，请使用cowboy_req:reply/4：
```
Req = cowboy_req:reply(200, #{
    <<"content-type">> => <<"text/plain">>
}, "Hello world!", Req0).
```

当响应具有正文时，您应始终设置content-type header。 但是，不需要设置内容长度标题; Cowboy自动完成。

响应主体和header值必须是二进制或iolist。 iolist是包含二进制文件，字符，字符串或其他iolist的列表。 这允许您从不同的部分构建响应，而无需进行任何连接：
```
Title = "Hello world!",
Body = <<"Hats off!">>,
Req = cowboy_req:reply(200, #{
    <<"content-type">> => <<"text/html">>
}, ["<html><head><title>", Title, "</title></head>",
    "<body><p>", Body, "</p></body></html>"], Req0).
```
这种构建响应的方法比连接更有效。 在幕后，列表中的每个元素都只是一个指针，这些指针在写入套接字时直接使用。

## Stream 回复

Cowboy提供了两个用于启动响应的函数，以及用于流式传输响应主体的附加功能。 Cowboy会在响应中添加任何必需的标题。

如果只需要设置状态代码，请使用cowboy_req:stream_reply / 2：
```
Req = cowboy_req:stream_reply(200, Req0),

cowboy_req:stream_body("Hello...", nofin, Req),
cowboy_req:stream_body("chunked...", nofin, Req),
cowboy_req:stream_body("world!!", fin, Req).
```

Cowboy_req:stream_body / 3的第二个参数指示此数据是否终止正文。 使用fin作为最终标志，否则使用nofin。

此代码段不会设置内容类型header。 不建议这样做。 所有具有正文的回复都应具有内容类型。 可以预先设置header，也可以使用cowboy_req:stream_reply / 3：
```
Req = cowboy_req:stream_reply(200, #{
    <<"content-type">> => <<"text/html">>
}, Req0),

cowboy_req:stream_body("<html><head>Hello world!</head>", nofin, Req),
cowboy_req:stream_body("<body><p>Hats off!</p></body></html>", fin, Req).
```

