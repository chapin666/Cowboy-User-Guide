# Request 对象

Req对象是一个变量，用于获取有关请求的信息，读取其正文或发送响应。

它实际上不是面向对象意义上的对象。 它是一个简单的映射，可以在从cowboy_req模块调用函数时直接访问或使用。

Req对象是几个不同章节的主题。 在本章中，我们将了解Req对象，并了解如何检索有关请求的信息。

## 直接访问

Req map包含许多记录的字段，可以直接访问。 它们是直接映射到HTTP的字段：请求方法; 使用的HTTP版本; 有效的URI组件方案，主机，端口，路径和qs; 请求header; 和连接对等地址和端口。

请注意，version字段可用于确定连接是否使用HTTP/2。

要访问字段，您只需在功能头中进行匹配即可。 以下示例发送一个简单的“Hello world！” 方法为GET时的响应，否则为405错误。

```
init(Req0=#{method := <<"GET">>}, State) ->
    Req = cowboy_req:reply(200, #{
        <<"content-type">> => <<"text/plain">>
    }, <<"Hello world!">>, Req0),
    {ok, Req, State};
init(Req0, State) ->
    Req = cowboy_req:reply(405, #{
        <<"allow">> => <<"GET">>
    }, Req0),
    {ok, Req, State}.
```

任何其他字段都是内部字段，不应被访问。 它们可能会在将来的版本中更改，包括维护版本，恕不另行通

除非严格必要，否则不建议在允许的情况下修改Req对象。 如果添加新字段，请确保命名字段名称，以便将来Cowboy更新或第三方项目不会发生冲突。

## cowboy_req 接口介绍

cowboy_req模块中的函数提供对请求信息的访问，但也提供了处理HTTP请求时常见的各种操作。

所有以动词开头的函数都表示动作。其他函数只返回相应的值（有时需要构建该值，但操作的成本等同于检索值）。

一些cowboy_req函数返回更新的Req对象。它们是读取，回复，设置和删除功能。忽略返回的Req不会导致其中某些行为不正确，强烈建议始终保留并使用最后返回的Req对象。 cowboy_req手册详细介绍了这些函数以及对Req对象进行了哪些修改。

一些对cowboy_req的调用有副作用。这是读取和回复功能的情况。Cowboy读取请求正文或在调用函数时立即回复。

如果出现问题，所有功能都会崩溃。通常无需捕获这些错误，Cowboy将根据崩溃发生的位置发送相应的4xx或5xx响应。

## Request 方法

可以直接获取请求方法：
```
#{method := Method} = Req.
```
或者使用一个函数
```
Method = cowboy_req:method(Req).
```
该方法是区分大小写的二进制字符串。 标准方法包括GET，HEAD，OPTIONS，PATCH，POST，PUT或DELETE。

## HTTP 版本

HTTP版本是信息性的。 它并不表示客户端很好地或完全地实现了协议。

通常不需要根据HTTP版本更改行为：Cowboy已经为您完成了。

但在某些情况下它可能很有用。 例如，可能希望重定向HTTP/1.1客户端以使用Websocket，而HTTP/2客户端继续使用HTTP/2。

可以直接检索HTTP版本：
```
#{version := Version} = Req.
```
或者直接使用一个函数：
```
Version = cowboy_req:version(Req).
```
Cowboy定义了'HTTP/1.0'，'HTTP/1.1'和'HTTP/2'版本。 自定义协议可以将自己的值定义为原子。

## 有效的请求URI

可以直接检索有效请求URI的方案，主机，端口，路径和查询字符串组件：
```
#{
    scheme := Scheme,
    host := Host,
    port := Port,
    path := Path,
    qs := Qs
} = Req.
```
或使用相关函数：
```
Scheme = cowboy_req:scheme(Req),
Host = cowboy_req:host(Req),
Port = cowboy_req:port(Req),
Path = cowboy_req:path(Req).
Qs = cowboy_req:qs(Req).
The scheme and host are lowercased case
```

该方案和主机是小写的不区分大小写的二进制字符串。 端口是表示端口号的整数。 路径和查询字符串是区分大小写的二进制字符串。

牛仔只定义了<<“http”>>和<<“https”>>方案。 选择它们是为了使安全HTTP/1.1或HTTP/2连接上的请求只有<<“https”>>。

可以使用cowboy_req:uri/1,2函数重建有效请求URI本身。 默认情况下，返回绝对URI：
```
%% scheme://host[:port]/path[?qs]
URI = cowboy_req:uri(Req).
```
Options可用于禁用或替换部分或全部组件。 可以通过这种方式生成各种URI或URI格式，包括原始形式：
```
%% /path[?qs]
URI = cowboy_req:uri(Req, #{host => undefined}).
```
协议相对形式：
```
%% //host[:port]/path[?qs]
URI = cowboy_req:uri(Req, #{scheme => undefined}).
```
没有查询字符串的绝对URI：
```
URI = cowboy_req:uri(Req, #{qs => undefined}).
```
另一个不同的host：
```
URI = cowboy_req:uri(Req, #{host => <<"example.org">>}).
```
和任何其他组合。

## 绑定

绑定是您在定义应用程序路由时选择提取的主机和路径组件。 它们仅在路由后可用。

Cowboy提供检索一个或所有绑定的函数。

要检索单个值：
```
Value = cowboy_req:binding(userid, Req).
```
尝试检索未绑定的值时，将返回undefined。 可以提供不同的默认值：
```
Value = cowboy_req:binding(userid, Req, 42).
```
要检索绑定的所有内容：
```
Bindings = cowboy_req:bindings(Req).
```

它们作为map返回，键是原子。

Cowboy路由器还允许您使用...限定符一次捕获许多主机或路径段。

主机名段：
```
HostInfo = cowboy_req:host_info(Req).
```
path段
```
PathInfo = cowboy_req:path_info(Req).
```
果...没有在route中使用，Cowboy将返回undefined。

## 查询 parameters

Cowboy提供了两种访问查询参数的功能。 您可以使用第一个来获取整个参数列表。
```
QsVals = cowboy_req:parse_qs(Req),
{_, Lang} = lists:keyfind(<<"lang">>, 1, QsVals).
```
Cowboy 只会解析查询字符串，而不会进行任何转换。 因此，此函数可能返回没有关联值的重复项或参数名称。 返回列表的顺序是未定义的。

当查询字符串为key=1＆key=2时，返回的列表将包含两个name key参数。

尝试使用PHP样式后缀[]时也是如此。 当查询字符串为key[]=1＆key[]=2时，返回的列表将包含两个名为key[]的参数。

当查询字符串只是键时，Cowboy将返回列表[{<<“key”>>，true}]，使用true表示参数键已定义，但没有值。

Cowboy提供的第二个功能允许您仅匹配您感兴趣的参数，同时使用约束进行所需的任何后期处理。 此函数返回一个map。
```
#{id := ID, lang := Lang} = cowboy_req:match_qs([id, lang], Req).
```
约束可以自动应用。 当id参数不是整数或lang参数为空时，以下代码段将崩溃。 同时，id的值将转换为整数项：
```
QsMap = cowboy_req:match_qs([{id, int}, {lang, nonempty}], Req).
```
还可以提供默认值。 如果找不到lang键，将使用默认值。 如果找到密钥但它具有空值，则不会使用它。
```
#{lang := Lang} = cowboy_req:match_qs([{lang, [], <<"en-US">>}], Req).
```

如果未提供缺省值且缺少值，则查询字符串将被视为无效，并且进程将崩溃。

当查询字符串是key=1＆key=2时，key的值将是列表[1,2]。 参数名称不需要包含PHP样式后缀。 可以使用约束来确保仅传递一个值。

## Headers

Header可以作为二进制字符串检索，也可以解析为更有意义的表示形式。

获得原始值：

```
HeaderVal = cowboy_req:header(<<"content-type">>, Req).
```

Cowboy希望所有标题名称都以小写二进制字符串形式提供。 无论基础协议如何，对于请求和响应都是如此。

当请求中缺少Header时，将返回undefined。 可以提供不同的默认值：
```
HeaderVal = cowboy_req:header(<<"content-type">>, Req, <<"text/plain">>).
```

可以直接检索所有标题：
```
#{headers := AllHeaders} = Req.
```

或者使用一个函数：
```
AllHeaders = cowboy_req:headers(Req).
```

Cowboy提供了解析单个标题的等效函数。 没有函数可以一次解析所有header。

要解析特定header：
```
ParsedVal = cowboy_req:parse_header(<<"content-type">>, Req).
```

如果它不知道如何解析给定header，或者该值无效，则抛出异常。 可以在手册中找到已知标题和默认值的列表。

缺少header时，返回undefined。 您可以更改默认值。 请注意，它应该是直接解析的值：
```
ParsedVal = cowboy_req:parse_header(<<"content-type">>, Req,
    {<<"text">>, <<"plain">>, []}).
```

## Peer

可以直接或使用函数检索连接的对等地址和端口号。

要直接检索对等方：
```
#{peer := {IP, Port}} = Req.
```
或者使用函数：
```
{IP, Port} = cowboy_req:peer(Req).
```
请注意，对等体对应于与服务器的连接的远程端，其可能是也可能不是客户端本身。 它也可以是代理或网关。