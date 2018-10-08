# 设计资源处理程序

本章旨在为您提供一个必须回答的问题列表，以便编写一个好的资源处理程序。 它可用作逐步指南。

## service

服务可以变得不可用，当它发生时，我们可以检测到它吗？ 例如，可以尽早检测到数据库连接问题。 我们也可能计划了全部或部分系统的中断。 实现service_available回调。

该服务实现了哪些HTTP方法？ 我们需要的不仅仅是标准OPTIONS，HEAD，GET，PUT，POST，PATCH和DELETE吗？ 我们根本不使用其中一种吗？ 实现known_methods回调。

## 资源处理程序的类型

我是为一组资源还是为一个资源编写处理程序？

每个语义都是完全不同的。 您不应该在同一个处理程序中混合集合和单个资源。

## 集合处理程序

如果您没有collection，请跳过此部分。

该集合是硬编码还是动态？例如，如果您使用route/users作为用户集合，则该集合是硬编码的;如果你使用/forums/:category来收集线程，那么它就不是。当集合被硬编码时，您可以安全地假设资源始终存在。

我应该采用哪些方法？

OPTIONS用于获取有关集合的一些信息。即使您没有实现它，也建议允许它，因为Cowboy内置了默认实现。

HEAD和GET用于检索集合。如果您允许GET，也允许使用HEAD，因为不需要额外的工作来使其工作。

POST用于在集合中创建新资源。在知道URI之前创建资源时，通过在集合上使用POST来创建资源非常有用，通常是因为它的一部分是动态生成的。常见的情况是某种自动递增的整数标识符。

很少允许使用下一种方法。

PUT用于创建新集合（当集合未硬编码时），或替换整个集合。

DELETE用于删除整个集合。

PATCH用于使用请求正文中给出的指令修改集合。 PATCH操作是原子的。 PATCH操作可以用于诸如重新排序之类的事情;添加，修改或删除部分集合。

## 单一资源处理程序

如果您遇到的是collection，请跳过此部分。

我应该采用哪些方法？

OPTIONS用于获取有关资源的一些信息。 即使您没有实现它，也建议允许它，因为Cowboy内置了默认实现。

HEAD和GET用于检索资源。 如果您允许GET，也允许使用HEAD，因为不需要额外的工作来使其工作。

POST用于更新资源。

PUT用于创建新资源（当它尚不存在时）或替换资源。

DELETE用于删除资源。

PATCH用于使用请求正文中给出的指令修改资源。 PATCH操作是原子的。 PATCH操作可以用于添加，移除或修改资源中的特定值。

## 资源

按照上面的讨论，实现allowed_methods回调。

资源是否始终存在？ 如果不是，请实现resource_exists回调。

我是否需要在客户端访问资源之前对其进行身份验证？ 我应该提供哪些认证机制？ 这可能包括基于表单，基于令牌（在URL或cookie中），HTTP基本，HTTP摘要，SSL证书或任何其他形式的身份验证。 实现is_authorized回调。

我需要细粒度的访问控制吗？ 我如何确定他们是授权访问？ 处理你的is_授权回调。

无论访问权限如何，都可以禁止访问资源？ 一个简单的例子是对资源的审查。 实现禁止回调。

资源URI的长度是否有任何限制？ 例如，URI可以用作存储中的密钥，并且可以具有长度限制。 实施uri_too_long。

## Representations

我提供哪些媒体类型？ 如果基于文本，提供什么字符集？ 我提供哪些语言？

实施强制性content_types_provided。 为了清楚起见，使用to_前缀回调。 例如，to_html或to_text。

如果适用，实现languages_provided或charsets_provided回调。

是否有任何其他标题可能使资源的表示变化？ 实现差异回调。

根据您对缓存内容的选择，您可能希望实现generate_etag，last_modified和expires回调中的一个或多个。

我是否希望用户或用户代理主动选择可用的表示？ 在响应正文中发送可用表示的列表并实现multiple_choices回调。

## 重定向

是否需要跟踪已删除的资源？ 例如，您可能有一种机制，其中移动资源会将重定向链接留给其新位置。 实现previous_existed回调。

资源是否被移动，是暂时的移动？ 如果它是显式临时的，例如由于维护，则实现moving_temporarily回调。 否则，实现moving_permanently回调。

## request

你需要读取查询字符串吗？ 自定义header？ 实现malformed_request并在此函数中执行所有解析和验证。 请注意，此时不应阅读正文。

请问有body吗？ 我知道它的大小吗？ 我愿意接受的请求body的最大规模是多少？ 实现valid_entity_length。

最后，看一下与您正在实现的方法相对应的部分。

## OPTIONS 方法

默认情况下，Cowboy会发回允许的方法列表。 我是否需要在响应中添加更多信息？ 实现选项方法。

## GET 和 HEAD 方法

如果实现方法GET和/或HEAD，则必须为content_types_provided回调返回的每个内容类型实现一个ProvideResource回调。

## PUT, POST 和 PATCH 方法

 如果实现PUT，POST和/或PATCH方法，则必须为它返回的每个内容类型实现content_types_accepted回调和一个AcceptCallback回调。 为了清楚起见，使用from_作为AcceptCallback回调名称的前缀。 例如，from_html或from_json。

我们是否希望允许POST方法直接通过其URI（如PUT）创建单个资源？ 实现allow_missing_post回调。 建议在这些情况下明确使用PUT。

使用PUT创建或替换资源时可能会发生冲突吗？ 我们是否要确保同一时间的两个更新不会相互抵消？ 实现is_conflict回调。

## DELETE 方法

如果实现方法DELETE，则必须实现delete_resource回调。

当delete_resource返回时，资源是否已从服务器中完全删除，包括来自任何缓存服务？ 如果没有，和/或如果删除是异步的，并且我们无法知道它已经完成，请实现delete_completed回调。

