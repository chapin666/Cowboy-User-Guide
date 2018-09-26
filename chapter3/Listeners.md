# Listeners

Listeners 是一组侦听端口以获取新连接的进程。 传入的连接由cowboy处理。 根据连接握手，可以使用一个或另一个协议。

本章专门针对cowboy。 有关侦听器的更多信息，请参阅[Ranch用户指南](https://ninenines.eu/docs/en/ranch/1.3/guide/listeners/)。

Cowboy提供两种类型的侦听器：一种侦听常用的TCP连接，另一种侦听 secure TLS连接。 它们都支持HTTP/1.1和HTTP/2协议。

## TCP listener

TCP侦听器将接受给定端口上的连接。 典型的HTTP服务器将在端口80上侦听。80端口在大多数平台上需要特殊权限，因此常见的替代方案是端口8080。

以下代码段启动一个侦听到8080端口上的连接：
```
start(_Type, _Args) ->
    Dispatch = cowboy_router:compile([
        {'_', [{"/", hello_handler, []}]}
    ]),
    {ok, _} = cowboy:start_clear(my_http_listener,
        [{port, 8080}],
        #{env => #{dispatch => Dispatch}}
    ),
    hello_erlang_sup:start_link().
```

“[入门](../chapter2/Getting_started.md)”一章使用了清晰的TCP侦听器。

在listener端口连接到Cowboy的客户端应该使用HTTP/1.1或HTTP/2。

Cowboy支持两种启动HTTP/2连接的方法：通过升级机制（[RFC 7540 3.2](https://tools.ietf.org/html/rfc7540#section-3.2)）或直接发送 preface（[RFC 7540 3.4](https://tools.ietf.org/html/rfc7540#section-3.4)）。

Cowboy的HTTP/1.1实现提供了与HTTP/1.0的兼容性。

## Secure TLS listener

安全TLS listener 将接受给定端口上的连接。 典型的HTTPS服务器将在端口443上侦听。端口443在大多数平台上需要特殊权限，因此常见的替代方案是端口8443。

Cowboy提供的功能将确保给出的TLS选项遵循HTTP/2 RFC的安全性。 例如，可以禁用某些TLS扩展或密码。 这也适用于此侦听器上的HTTP/1.1连接。 如果不希望这样，可以直接使用Ranch来设置自定义监听器。

```
start(_Type, _Args) ->
    Dispatch = cowboy_router:compile([
        {'_', [{"/", hello_handler, []}]}
    ]),
    {ok, _} = cowboy:start_tls(my_http_listener,
        [
            {port, 8443},
            {certfile, "/path/to/certfile"},
            {keyfile, "/path/to/keyfile"}
        ],
        #{env => #{dispatch => Dispatch}}
    ),
    hello_erlang_sup:start_link().
```

在安全监听器上连接到 Cowboy 的客户端应使用ALPN TLS扩展来指示他们理解的协议。 当两者都受支持时，Cowboy总是优先于HTTP/1.1上的HTTP/2。 当客户端既不支持，也不缺少ALPN扩展时，Cowboy期望使用HTTP/1.1。

Cowboy还通过较旧的NPN TLS扩展来宣传HTTP/2支持以实现兼容性。 但请注意，当Cowboy 2.0发布时，默认情况下可能无法启用此支持。

Cowboy的HTTP/1.1实现提供了与HTTP/1.0的兼容性。

## 协议配置

HTTP/1.1和HTTP/2协议共享相同的语义; 只有他们的框架不同。 第一个是文本协议，第二个是二进制协议。

Cowboy不会分离HTTP/1.1和HTTP/2的配置。 一切都进入同一张map。 许多选项都是共享的。