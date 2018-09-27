# 静态文件

Cowboy带有一个随时可用的处理程序来提供静态文件。 它是为了在开发期间提供文件提供方便。

对于生产中的系统，请考虑使用市场上可用的众多内容分发网络（CDN）之一，因为它们是提供文件的最佳解决方案。

静态处理程序可以提供给定目录中的一个文件或所有文件。 可以配置etag生成和mime类型。

##  提供一个文件

您可以使用静态处理程序从应用程序的专用目录中提供一个特定文件。 例如，当客户端请求/path时，这对于提供index.html文件特别有用。 配置的路径相对于给定应用程序的专用目录。

每当访问路径/时，以下规则将从应用程序my_app的priv目录中提供static/index.html文件：
```
{"/", cowboy_static, {priv_file, my_app, "static/index.html"}}
```
您还可以指定文件的绝对路径，或相对于当前目录的文件路径：
```
{"/", cowboy_static, {file, "/var/www/index.html"}}
```

## Serve all files from a directory

您还可以使用静态处理程序来提供可在配置目录中找到的所有文件。 处理程序将使用path_info信息来解析文件位置，这意味着您的路由必须以[...]模式结束才能工作。 提供所有文件，包括可在子文件夹中找到的文件。

您可以指定相对于应用程序私有目录的目录。

当请求的路径以/assets/开头时，以下规则将为static/assets文件夹中的应用程序my_app的priv目录中的任何文件提供服务：
```
{"/assets/[...]", cowboy_static, {priv_dir, my_app, "static/assets"}}
```
您还可以指定目录的绝对路径或相对于当前目录设置它：
```
{"/assets/[...]", cowboy_static, {dir, "/var/www/assets"}}
```
## 自定义mimetype检测

默认情况下，Cowboy会通过查看扩展名来尝试识别静态文件的mimetype。

您可以覆盖计算静态文件的mimetype的函数。 当Cowboy缺少您需要处理的mimetype时，或者当您想要减少列表以使查找更快时，它会很有用。 您还可以提供将无条件使用的硬编码mimetype。

函数内置两个功能。 默认函数仅处理构建Web应用程序时使用的常见文件类型。 另一个函数是数百种mimetypes的广泛列表，几乎可以满足您的任何需求。 你当然可以创建自己的函数。

要使用默认函数，您不必配置任何内容，因为它是默认设置。 但是，如果你坚持，以下将完成这项工作：
```
{"/assets/[...]", cowboy_static, {priv_dir, my_app, "static/assets",
    [{mimetypes, cow_mimetypes, web}]}}
```

您所见，有一个可选字段可能包含一个较少使用的选项列表，如mimetypes或etag。 所有选项类型都有此可选字段。

要使用几乎可以检测任何mimetype的函数，以下配置将执行：
```
{"/assets/[...]", cowboy_static, {priv_dir, my_app, "static/assets",
    [{mimetypes, cow_mimetypes, all}]}}
```

你现在可能已经注意到了这种模式。 配置需要模块和函数名称，因此您可以使用任何自己的函数：
```
{"/assets/[...]", cowboy_static, {priv_dir, my_app, "static/assets",
    [{mimetypes, Module, Function}]}}
```
执行mimetype检测的函数接收单个参数，该参数是磁盘上文件的路径。 建议以元组形式返回mimetype，尽管也允许使用二进制字符串（但需要额外处理）。 如果函数无法找出mimetype，那么它应该返回{<<"application">>，<<"octet-stream">>，[]}。

当静态处理程序无法找到扩展名时，它将把文件作为application/octet-stream发送。 接收此类文件的浏览器将尝试将其直接下载到磁盘。

最后，mimetype可以对所有文件进行硬编码。 这与文件和priv_file选项结合使用特别有用，因为它避免了不必要的计算：
```
{"/", cowboy_static, {priv_file, my_app, "static/index.html",
    [{mimetypes, {<<"text">>, <<"html">>, []}}]}}
```

## 生成etag

默认情况下，静态处理程序将根据大小和修改时间生成etag标头值。 但是，此解决方案不能应用于所有系统。 例如，它会在节点集群上执行得相当糟糕，因为文件元数据会因服务器而异，在每台服务器上提供不同的etag。

但是，您可以更改etag的计算方式：
```
{"/assets/[...]", cowboy_static, {priv_dir, my_app, "static/assets",
    [{etag, Module, Function}]}}
```
此函数将接收三个参数：磁盘上文件的路径，文件大小和上次修改时间。 在分布式设置中，通常使用文件路径来检索所有服务器上相同的etag值。

您还可以完全禁用etag处理：
```
{"/assets/[...]", cowboy_static, {priv_dir, my_app, "static/assets",
    [{etag, false}]}}
```