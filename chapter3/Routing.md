# Routing

Cowboy默认不做任何事情。

要使Cowboy有用，您需要将URI映射到将处理请求的Erlang模块。 这称为路由。

当Cowboy收到请求时，它会尝试匹配请求的主机和路径到配置的路由。 当匹配时，执行路由的关联处理程序。

路由需要在Cowboy使用之前进行编译。 编译的结果是调度规则。

### 语法

路由的结构一般定义如下。
```
Routes = [Host1, Host2, ... HostN].
```
每个主机包含主机的匹配规则以及可选约束，以及路径组件的路由列表。
```
Host1 = {HostMatch, PathsList}.
Host2 = {HostMatch, Constraints, PathsList}.
```
路径组件的路由列表定义类似于主机列表。
```
PathsList = [Path1, Path2, ... PathN].
```
最后，每个路径包含路径的匹配规则以及可选约束，并为我们提供了与其初始状态一起使用的处理程序模块。
```
Path1 = {PathMatch, Handler, InitialState}.
Path2 = {PathMatch, Constraints, Handler, InitialState}.
```
继续阅读以了解有关匹配语法和可选约束的更多信息。

### 匹配语法

匹配语法用于将主机名和路径与其各自的处理程序相关联。

主机和路径的匹配语法相同，但有一些细微之处。 实际上，段分隔符是不同的，并且主机从最后一个段到第一个段开始匹配。 所有示例都将包含主机和路径匹配规则，并解释遇到的差异。

排除我们将在本节末尾解释的特殊值，最简单的匹配值是主机或路径。 它可以作为string()或bianry()给出。
```
PathMatch1 = "/".
PathMatch2 = "/path/to/resource".

HostMatch1 = "cowboy.example.org".
```

如您所见，以这种方式定义的所有路径必须以斜杠字符开头。 请注意，就路由而言，这两个路径是相同的。
```
PathMatch2 = "/path/to/resource".
PathMatch3 = "/path/to/resource/".
```

带有和不带尾随点的主机等同于路由。 类似地，具有和不具有前导点的主机也是等效的。
```
HostMatch1 = "cowboy.example.org".
HostMatch2 = "cowboy.example.org.".
HostMatch3 = ".cowboy.example.org".
```

可以提取主机和路径的段，并将值存储在Req对象中供以后使用。 我们将这些值称为绑定。

绑定的语法非常简单。 以：字符开头的段表示在段结束之后的内容是将存储段值的绑定的名称。
```
PathMatch = "/hats/:name/prices".
HostMatch = ":subdomain.example.org".
```
如果这两个在路由时最终匹配，则最终会定义两个绑定，子域和名称，每个绑定包含定义它们的段值。 例如，URL http://test.example.org/hats/wild_cowboy_legendary/prices将导致将值测试绑定到名称子域，并将值wild_cowboy_legendary绑定到名称。 稍后可以使用cowboy_req:binding/{2,3}检索它们。 绑定名称必须以原子形式给出。

您可以使用特殊的绑定名称来模仿Erlang中的下划线变量。 与_绑定的任何匹配都将成功，但数据将被丢弃。 这对于一次性匹配多个域名特别有用。
```
HostMatch = "ninenines.:_".
```
同样，可以有可选的段。 括号内的任何内容都是可选的。
```
PathMatch = "/hats/[page/:number]".
HostMatch = "[www.]ninenines.eu".
```
您还可以使用叠加的可选段。
```
PathMatch = "/hats/[page/[:number]]".
```

您可以使用[...]检索主机或路径的其余部分。 在主机的情况下，它将匹配之前的任何内容，在路径的情况下，在先前匹配的段之后的任何内容。 这是可选段的特殊情况，因为它可以包含零个，一个或多个段。 然后，您可以分别使用cowboy_req:host_info/1和cowboy_req:path_info/1查找段。 它们将表示为段列表。
```
PathMatch = "/hats/[...]".
HostMatch = "[...]ninenines.eu".
```
如果绑定在路由规则中出现两次，则匹配仅在共享相同值时才会成功。 这将复制Erlang模式匹配行为。
```
PathMatch = "/hats/:name/:name".
```
当存在可选段时也是如此。 在这种情况下，仅当段可用时，两个值必须相同。
```
PathMatch = "/:user/[...]".
HostMatch = ":user.github.com".
```
最后，可以使用两个特殊匹配值。 第一个是原子'_'，它将匹配任何主机或路径。
```
PathMatch = '_'.
HostMatch = '_'.
```

第二个是特殊主机匹配“*”，它将匹配通配符路径，通常与OPTIONS方法一起使用。
```
HostMatch = "*".
```

### Constraints

匹配完成后，可以针对一组约束测试生成的绑定。 只有在定义绑定时才会测试约束。 它们按照您定义的顺序运行。 只有在全部成功的情况下，匹配才会成功。 如果匹配失败，那么Cowboy将尝试列表中的下一个路线。

用于约束的格式与cowboy_req中的匹配函数相同：它们作为可能具有一个或多个约束的字段列表提供。 虽然路由器接受相同的格式，但它将跳过没有约束的字段，并且还会忽略默认值（如果有）。

阅读有关[约束](https://ninenines.eu/docs/en/cowboy/2.4/guide/constraints/)的更多信息

## Compilation

必须先编译路由，然后cowboy才能使用它们。 编译步骤规范化路由以简化代码并加快执行速度，但最终仍然逐个查找路由。 更快的编译策略可能是将路由直接编译为Erlang代码，但需要更多的依赖。

要编译路由，只需调用相应的函数：
```
Dispatch = cowboy_router:compile([
    %% {HostMatch, list({PathMatch, Handler, InitialState})}
    {'_', [{'_', my_handler, #{}}]}
]),
%% Name, NbAcceptors, TransOpts, ProtoOpts
cowboy:start_clear(my_http_listener,
    [{port, 8080}],
    #{env => #{dispatch => Dispatch}}
).
```

## Live update

您可以使用cowboy:set_env/3函数来更新路由使用的调度列表。 这将适用于侦听器接受的所有新连接：
```
Dispatch = cowboy_router:compile(Routes),
cowboy:set_env(my_http_listener, dispatch, Dispatch).
```

请注意，您需要在更新之前再次编译路由。