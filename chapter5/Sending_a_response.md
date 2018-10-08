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

Cowboy_req:stream_body/3的第二个参数指示此数据是否终止正文。 使用fin作为最终标志，否则使用nofin。

此代码段不会设置内容类型header。 不建议这样做。 所有具有正文的回复都应具有内容类型。 可以预先设置header，也可以使用cowboy_req:stream_reply/3：
```
Req = cowboy_req:stream_reply(200, #{
    <<"content-type">> => <<"text/html">>
}, Req0),

cowboy_req:stream_body("<html><head>Hello world!</head>", nofin, Req),
cowboy_req:stream_body("<body><p>Hats off!</p></body></html>", fin, Req).
```

HTTP提供了一些流式响应body的不同方式。 Cowboy将根据HTTP版本以及请求和响应标头选择最合适的一个。

虽然不是必需的，但如果您事先知道，建议您在响应中设置content-length标头。 这将确保选择最佳响应方法，并帮助客户了解何时完全接收响应。

Cowboy还提供发送响应预告片的功能。 响应预告片在语义上等同于您在响应中发送的标头，只有它们在最后发送。 这对于将信息附加到响应之前特别有用，该响应在完全生成响应主体之前无法生成。

预告片字段必须列在预告片标题中。 客户或中间人可能会删除未列出的任何字段。

```
Req = cowboy_req:stream_reply(200, #{
    <<"content-type">> => <<"text/html">>,
    <<"trailer">> => <<"expires, content-md5">>
}, Req0),

cowboy_req:stream_body("<html><head>Hello world!</head>", nofin, Req),
cowboy_req:stream_body("<body><p>Hats off!</p></body></html>", nofin, Req),

cowboy_req:stream_trailers(#{
    <<"expires">> => <<"Sun, 10 Dec 2017 19:13:47 GMT">>,
    <<"content-md5">> => <<"c6081d20ff41a42ce17048ed1c0345e2">>
}, Req).
```

流以拖车结束。 发送预告片后不再可能发送数据。 在流式传输body时设置fin标志后，您无法发送预告片。

## 预设响应标头

Cowboy提供了设置响应头的功能，而无需立即发送它们。 它们存储在Req对象中，并在调用reply函数时作为响应的一部分发送。

要设置响应标头：
```
Req = cowboy_req:set_resp_header(<<"allow">>, "GET", Req0).
```
标题名称必须是小写二进制文件。

不要使用此功能来设置cookie。 有关更多信息，请参阅Cookie一章。

要检查是否已设置响应标头：
```
cowboy_req:has_resp_header(<<"allow">>, Req).
```

如果header已设置，则返回true，否则返回false。

要删除先前设置的响应标头：
```
Req = cowboy_req:delete_resp_header(<<"allow">>, Req0).
```

## headers 概述

由于Cowboy提供了设置响应标头和body的不同方法，因此可能会发生冲突，因此了解标头设置两次时会发生什么情况非常重要。

headers来自五个不同的起源：

- 特定于协议的header（例如HTTP/1.1的连接头）
- 其他必需的header（例如日期header）
- 预设header
- 给回复函数的header
- 设置cookie header

Cowboy不允许覆盖特定于协议的标头。

在发送响应之前，Set-cookie header将始终附加在header列表的末尾。

给予回复功能的header将始终覆盖预设header和所需header。 如果在其中的两个或三个中找到header，则选择回复函数中的header，然后删除其他header。

同样，预设header将始终覆盖所需的header。

为了说明，请查看以下代码段。 默认情况下，Cowboy会使用值“Cowboy”发送服务器header。 我们可以覆盖它：
```
Req = cowboy_req:reply(200, #{
    <<"server">> => <<"yaws">>
}, Req0).
```

## 预设响应体

Cowboy提供了设置响应体的函数，而无需立即发送它。 它存储在Req对象中，并在调用reply函数时发送。

要设置响应正文：
```
Req = cowboy_req:set_resp_body("Hello world!", Req0). 
```
要检查是否已设置响应正文：
```
cowboy_req:has_resp_body(Req).
```

如果正文已设置且非空，则返回true，否则返回false。

仅当使用的回复函数是cowboy_req:reply/2或cowboy_req:reply/3时，才会发送预设的响应正文。

## 发送文件

Cowboy提供了发送文件的快捷方式。 使用cowboy_req:reply/4时，或者在预设响应头时，可以给Cowboy一个sendfile元组：
```
{sendfile, Offset, Length, Filename}
```
根据Offset或Length的值，可以发送整个文件，也可以只发送一部分文件。

即使发送整个文件，也需要长度。 Cowboy在内容长度标题中发送它。

在回复时发送文件：
```
Req = cowboy_req:reply(200, #{
    <<"content-type">> => "image/png"
}, {sendfile, 0, 12345, "path/to/logo.png"}, Req0).
```

## 信息回复

Cowboy允许您发送信息响应。

信息响应是状态代码在100和199之间的响应。可以在正确响应之前发送任何数字。 发送信息响应不会改变正确响应的行为，并且客户端应忽略他们不理解的任何信息响应。

以下代码段发送103信息响应，其中包含一些预期在最终响应中的标头。
```
Req = cowboy_req:inform(103, #{
    <<"link">> => <<"</style.css>; rel=preload; as=style, </script.js>; rel=preload; as=script">>
}, Req0).
```

## Push

HTTP/2协议引入了推送与响应中发送的资源相关的资源的能力。 Cowboy为此提供了两个功能：cowboy_req:push/3,4。

推送仅适用于HTTP/2。 如果协议不支持推送请求，Cowboy将自动忽略推送请求。

必须在任何回复函数之前调用push函数。 否则将导致崩溃。

要推送资源，您需要提供与执行请求的客户端相同的信息。 这包括HTTP方法，URI和任何必要的请求标头。

Cowboy默认只要求您提供资源和请求标头的路径。 URI的其余部分取自当前请求（不包括查询字符串，设置为空），默认情况下该方法为GET。

以下代码段推送在响应中链接到的CSS文件：
```
owboy_req:push("/static/style.css", #{
    <<"accept">> => <<"text/css">>
}, Req0),
Req = cowboy_req:reply(200, #{
    <<"content-type">> => <<"text/html">>
}, ["<html><head><title>My web page</title>",
    "<link rel='stylesheet' type='text/css' href='/static/style.css'>",
    "<body><p>Welcome to Erlang!</p></body></html>"], Req0).
```

要覆盖方法，scheme，host，port或query字符串，只需传入第四个参数。 以下代码段使用不同的主机名：
```
cowboy_req:push("/static/style.css", #{
    <<"accept">> => <<"text/css">>
}, #{host => <<"cdn.example.org">>}, Req),
```

推送资源不一定是文件。 只要推送请求是可缓存的，安全的并且不包括正文，就可以推送资源。

在引擎下，Cowboy处理推送请求与正常请求相同：创建了一个不同的进程，最终将向客户端发送响应。