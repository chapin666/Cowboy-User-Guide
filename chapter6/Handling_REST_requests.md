# 处理 REST 请求

REST在Cowboy中作为子协议实现。 请求作为状态机处理，其中包含许多可选回调，用于描述资源和修改计算机的行为。

REST处理程序是处理HTTP请求的推荐方法。

## Initialization

首先，调用init/2回调。 此回调对所有处理程序都是通用的。 要将REST用于当前请求，此函数必须返回cowboy_rest元组。
```
init(Req, State) ->
    {cowboy_rest, Req, State}.
```

Cowboy然后将切换到REST协议并开始执行状态机。

到达流程结束后，如果已定义，将调用terminate/3回调。

## Methods

REST组件具有用于处理以下HTTP方法的代码：HEAD，GET，POST，PATCH，PUT，DELETE和OPTIONS。

可以接受其他方法，但是它们目前没有为它们定义特定的回调。

## Callbacks

所有回调都是可选的。有些可能会成为强制性的，具体取决于其他定义的回调返回。下一章中的各种流程图对于确定您需要哪些回调非常有用。

所有回调都有两个参数，Req对象和State，并返回{Value，Req，State}形式的三元素元组。

几乎所有的回调也可以返回{stop，Req，State}来停止执行请求，并{{switch_handler，Module}，Req，State}或{{switch_handler，Module，Opts}，Req，State}切换到a不同的处理程序类例外是expires generate_etag，last_modified和variances。

下表总结了回调及其默认值。如果未定义回调，则将使用默认值。请查看流程图以找出每个返回值的结果。

在下表中，“skip”表示如果未定义回调，则完全跳过回调，直接转到下一步。同样，“none”表示此回调没有默认值。

Callback name	|Default value|
--------------------|-------
allowed_methods|	[<<"GET">>, <<"HEAD">>, <<"OPTIONS">>]|
allow_missing_post|	true|
charsets_provided|	skip|
content_types_accepted|	none
content_types_provided|	[{{ <<"text">>, <<"html">>, '*'}, to_html}]
delete_completed|	true
delete_resource	|false
expires	|undefined
forbidden|	false
generate_etag	|undefined
is_authorized	|true
is_conflict	|false
known_methods |	[<<"GET">>, <<"HEAD">>, <<"POST">>, <<"PUT">>, <<"PATCH">>, <<"DELETE">>, <<"OPTIONS">>]
languages_provided|	skip
last_modified	|undefined
malformed_request|	false
moved_permanently	|false
moved_temporarily	|false
multiple_choices	|false
options	|ok
previously_existed|false
resource_exists|true
service_available|true
uri_too_long|false
valid_content_headers|true
valid_entity_length	|true
variances	|[]

正如您所看到的，Cowboy试图通过使用深思熟虑的默认值尽可能地继续请求。

除此之外，还可以通过content_types_accepted/2和content_types_provided/2指定任意数量的用户定义回调。 它们可以采用任何名称，但建议为每个函数的回调使用单独的前缀。 例如，from_html和to_html在第一种情况下指示我们接受以HTML格式给出的资源，在第二种情况下我们将其作为HTML发送。

## Meta data

Cowboy会在执行的各个点为Req对象设置信息值。 您可以通过直接匹配Req对象来检索它们。 值在下表中定义：
 Key |	Details
|--------|--------|
media_type|	The content-type negotiated for the response entity.
language|	The language negotiated for the response entity.
charset |	The charset negotiated for the response entity.

它们可用于发送一个正确的正文，其中包含对使用HEAD或GET以外的方法的请求的响应。

## Response headers

Cowboy将在执行REST代码时自动设置响应标头。 它们列在下表中。

Header name |	Details
|--------|--------|
content-language|	Language used in the response body
content-type|	Media type and charset of the response body
etag |	Etag of the resource
expires|	Expiration date of the resource
last-modified	|Last modification date for the resource
location |	Relative or absolute URI to the requested resource
vary|	List of headers that may change the representation of the resource
