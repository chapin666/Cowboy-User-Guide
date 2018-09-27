# 入门

Erlang不仅仅是一种语言，它还是一个适用于您的应用程序的操作系统。 Erlang开发人员很少编写独立模块，他们编写库或应用程序，然后将它们捆绑到所谓的版本中。 版本包含Erlang VM以及运行节点所需的所有应用程序，因此可以直接将其推送到生产环境。

本章将指导您完成设置Cowboy，编写第一个应用程序以及生成第一个应用程序的所有步骤。 在本章的最后，您应该了解将第一个Cowboy应用程序推向生产所需的一切。

## 先决条件

我们将使用[Erlang.mk](https://github.com/ninenines/erlang.mk)构建系统。 如果您使用的是Windows，请在继续之前查看[安装说明](https://erlang.mk/guide/installation.html)以获取环境设置。

## Bootstrap

首先，让我们为我们的应用程序创建目录。
```
$ mkdir hello_erlang
$ cd hello_erlang
```
然后我们需要下载Erlang.mk。 使用以下命令或手动下载。
```
$ wget https://erlang.mk/erlang.mk
```
我们现在可以引导我们的应用程序了。 由于我们要生成一个版本，我们也会同时引导它。
```
$ make -f erlang.mk bootstrap bootstrap-rel
```
这将创建一个Makefile，一个基本应用程序以及创建该版本所需的发布文件。 我们已经可以构建并启动此版本。
```
$ make run
...
(hello_erlang@127.0.0.1)1>
```

输入命令i()。 将显示正在运行的进程，包括一个名为hello_erlang_sup的进程。 这是我们应用的supervisor。

该版本目前无效。 在本章的其余部分，我们将添加Cowboy作为依赖，并编写一个简单的“Hello world！”处理程序。

## Cowboy 安装

我们将修改Makefile来告诉构建系统它需要获取和编译Cowboy：
```
PROJECT = hello_erlang

DEPS = cowboy
dep_cowboy_commit = 2.4.0

DEP_PLUGINS = cowboy

include erlang.mk
```

我们还告诉构建系统加载Cowboy提供的插件。 这些包括我们即将使用的预定义模板。

如果你现在运行，Cowboy将被包含在发行版中并自动启动。 但这还不够，因为Cowboy默认情况下不会做任何事情。 我们仍然需要告诉Cowboy监听连接。

## 监听连接

首先，我们定义Cowboy用于将请求映射到处理程序模块的路由，然后我们启动侦听器。 这最好在应用程序启动时完成。

打开src/hello_erlang_app.erl文件并将必要的代码添加到start/2函数中，使其如下所示：
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

[路由章节](https://ninenines.eu/docs/en/cowboy/2.4/guide/routing)中详细说明了路由。 在本教程中，我们将路径/映射到处理程序模块hello_handler。 该模块尚不存在。

构建并启动发布，然后在浏览器中打开http://localhost:8080。 您将收到500错误，因为模块丢失。 任何其他URL，如http://localhost:8080/test，都将导致404错误。

## 处理请求

Cowboy具有不同类型的处理程序，包括REST和Websocket处理程序。 在本教程中，我们将使用普通的HTTP处理程序。

从模板生成处理程序：
```
$ make new t=cowboy.http n=hello_handler
```
然后，打开src/hello_handler.erl文件并修改这样的init/2函数以发送回复。
```
init(Req0, State) ->
    Req = cowboy_req:reply(200,
        #{<<"content-type">> => <<"text/plain">>},
        <<"Hello Erlang!">>,
        Req0),
    {ok, Req, State}.
```

响应体设置为text/plain和Hello Erlang！。

如果您运行该版本并在浏览器中打开http://localhost:8080，您应该得到一个不错的Hello Erlang！显示！