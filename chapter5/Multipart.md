# Multipart 请求

Multipart源自MIME，这是一种扩展电子邮件格式的Internet标准。

多部分消息是部件列表。 部件包含标题和正文。 部件的主体可以是任何介质类型，并包含文本或二进制数据。 部件可能包含多部分介质类型。

在HTTP的上下文中，multipart最常用于multipart/form-data媒体类型。 这是浏览器用于通过HTML表单上传文件的内容。

multipart/byteranges也很常见。 它是用于从资源发送任意字节的媒体类型，使客户端能够恢复下载。

## Form-data

在正常情况下，当提交表单时，浏览器将使用application/x-www-form-urlencoded content-type。 此类型只是键和值的列表，因此不适合上传文件。

这就是multipart/form-data内容类型的来源。当表单配置为使用此内容类型时，浏览器将创建一个多部分消息，其中每个部分对应于表单上的字段。对于文件，它还会在部件标题中添加一些元数据，例如文件名。

具有文本输入，文件输入和选择框的表单将产生具有三个部分的多部分消息，每个部分对应一个字段。

浏览器尽力确定以这种方式发送的文件的媒体类型，但您不应该依赖它来确定文件的内容。 建议对内容进行适当的调查。

## 检查多部分消息

content-type标头指示存在多部分消息：
```
{<<"multipart">>, <<"form-data">>, _}
    = cowboy_req:parse_header(<<"content-type">>, Req).
```

## 读取多部分消息

Cowboy提供了两组函数，用于将请求体读取为多部分消息。

cowboy_req:read_part/1,2函数返回下一部分的标题，如果有的话。

cowboy_req:read_part_body/1,2函数返回当前部分的主体。 对于大型机构，您可能需要多次调用该函数。

要读取多部分消息，您需要迭代其所有部分：
```
multipart(Req0) ->
    case cowboy_req:read_part(Req0) of
        {ok, _Headers, Req1} ->
            {ok, _Body, Req} = cowboy_req:read_part_body(Req1),
            multipart(Req);
        {done, Req} ->
            Req
    end.
```

当请求体太大时，Cowboy会返回一个更多的元组，并让你循环直到零件体被完全读取。

函数cow_multipart:form_data/1可用于从multipart/form-data消息中快速获取有关部件的信息。 该函数返回数据或文件元组，具体取决于这是正常字段还是正在上载的文件。

以下代码段将使用此函数并根据零件是否为文件使用不同的策略：
```
multipart(Req0) ->
    case cowboy_req:read_part(Req0) of
        {ok, Headers, Req1} ->
            Req = case cow_multipart:form_data(Headers) of
                {data, _FieldName} ->
                    {ok, _Body, Req2} = cowboy_req:read_part_body(Req1),
                    Req2;
                {file, _FieldName, _Filename, _CType} ->
                    stream_file(Req1)
            end,
            multipart(Req);
        {done, Req} ->
            Req
    end.

stream_file(Req0) ->
    case cowboy_req:read_part_body(Req0) of
        {ok, _LastBodyChunk, Req} ->
            Req;
        {more, _BodyChunk, Req} ->
            stream_file(Req)
    end.
```

零件标题和正文阅读功能都可以获取将提供给请求正文阅读功能的选项。 默认情况下，cowboy_req:read_part/1最多可读取64KB，最长可达5秒。 cowboy_req:read_part_body/1与cowboy_req:read_body/1具有相同的默认值。

要更改零件标题的默认值：
```
Cowboy_req:read_part(Req, #{length => 128000}).
```
对于部分请求体：
```
cowboy_req:read_part_body(Req, #{length => 1000000, period => 7000}).
```

## 跳过不需要的部分

不必阅读部分body。 当你请求下一部分的body时，Cowboy会自动跳过它。

以下代码段会读取所有标题并跳过所有正文：
```
multipart(Req0) ->
    case cowboy_req:read_part(Req0) of
        {ok, _Headers, Req} ->
            multipart(Req);
        {done, Req} ->
            Req
    end.
```
同样，如果你开始阅读body并且它最终太大，你可以继续下一部分。 Cowboy会自动跳过剩下的东西。

虽然Cowboy可以自动跳过部分主体，但读取速率是不可配置的。 根据您的应用程序，您可能希望手动跳过，特别是如果您在跳过时观察到性能不佳。

您也不必阅读所有部分。 您可以在找到所需数据后立即停止阅读。