# 使用 cookies

Cookie是一种允许应用程序在无状态HTTP协议之上维护状态的机制。

Cookie是key/value存储，其中key和value以纯文本格式存储。它们会在延迟或浏览器关闭后过期。它们可以在特定域名或路径上配置，并限制为安全资源（通过HTTPS发送或下载），或仅限于服务器（禁止从客户端脚本访问）。

Cookie key事实上区分大小写。

Cookie存储在客户端，并随每个后续请求一起发送，这些请求与存储它们的域和路径相匹配，直到它们过期。这可以产生不可忽视的成本。

Cookie不应被视为安全。它们以纯文本形式存储在用户的计算机上，可以被任何程序读取。使用清除连接时，代理也可以读取它们。始终在使用前验证该值，并且绝不会在其中存储任何敏感信息。

服务器设置的Cookie仅在客户端接收包含它们的响应后的请求中可用。

Cookie可能会重复发送。这通常有助于更新到期时间并避免丢失cookie。

## 设置Cookie

默认情况下，Cookie会在会话期间定义：
```
SessionID = generate_session_id(),
Req = cowboy_req:set_resp_cookie(<<"sessionid">>, SessionID, Req0).
```
它们也可以设置为几秒钟的持续时间：
```
SessionID = generate_session_id(),
Req = cowboy_req:set_resp_cookie(<<"sessionid">>, SessionID, Req0,
    #{max_age => 3600}).
```
要删除cookie，请将max_age设置为0：
```
SessionID = generate_session_id(),
Req = cowboy_req:set_resp_cookie(<<"sessionid">>, SessionID, Req0,
    #{max_age => 0}).
```
要将cookie限制到特定的域和路径，可以使用相同名称的选项：
```
Req = cowboy_req:set_resp_cookie(<<"inaccount">>, <<"1">>, Req0,
    #{domain => "my.example.org", path => "/account"}).
```

Cookie将与此域及其所有子域以及此路径上的资源或路径层次结构中的更深层的请求一起发送。

要将Cookie限制为安全通道（通常是通过HTTPS提供的资源）：
```
SessionID = generate_session_id(),
Req = cowboy_req:set_resp_cookie(<<"sessionid">>, SessionID, Req0,
    #{secure => true}).
```

防止客户端脚本访问cookie：
```
SessionID = generate_session_id(),
Req = cowboy_req:set_resp_cookie(<<"sessionid">>, SessionID, Req0,
    #{http_only => true}).
```

Cookie也可以在客户端设置，例如使用Javascript。

## Cookie 读取

客户端只发回cookie名称和值。 可以设置的所有其他选项永远不会被发回。

Cowboy提供两种阅读cookie的功能。 两者都涉及解析cookie头，因此不应重复调用。

您可以将所有cookie作为键/值列表：
```
Cookies = cowboy_req:parse_cookies(Req),
{_, Lang} = lists:keyfind(<<"lang">>, 1, Cookies).
```

或者，您可以对cookie执行匹配并仅检索所需的匹配，同时使用约束进行任何所需的后处理。 此函数返回一个映射：
```
#{id := ID, lang := Lang} = cowboy_req:match_cookies([id, lang], Req).
```

您可以使用约束来验证值，同时匹配它们。 如果id cookie不是整数，或者lang cookie为空，则以下代码段将崩溃。 此外，id cookie值将转换为整数术语：
```
CookiesMap = cowboy_req:match_cookies([{id, int}, {lang, nonempty}], Req).
```
请注意，如果两个cookie共享相同的名称，则映射值将是两个cookie值的列表。

可以提供默认值。 如果找不到lang cookie，将使用默认值。 如果找到cookie但它具有空值，则不会使用它：
```
#{lang := Lang} = cowboy_req:match_cookies([{lang, [], <<"en-US">>}], Req).
```

如果未提供缺省值且缺少值，则抛出异常。