# 介绍

Coboy是Erlang/OTP的小型，快速和现代的HTTP服务器。

Cowboy旨在提供完整的现代Web堆栈。 这包括HTTP/1.1，HTTP/2，Websocket，服务器发送事件和基于Web机器的REST。

Cowboy具有内省和跟踪功能，使开发人员能够随时准确了解发生的情况。 其模块化设计还可以轻松地使开发人员添加仪器。

Coboy是一个高质量的项目。 它具有较小的代码库，非常高效（在延迟和内存使用方面）并且可以轻松嵌入到另一个应用程序中。

Coboy是干净的Erlang代码。 它包括数百个测试，其代码完全符合Dialyzer。 它也有详细记录，并具有功能参考，用户指南和众多教程。

## 先决条件

建议您阅读初学者Erlang知识以阅读本指南。

建议您了解HTTP协议，但不是必需的，因为它将在整个指南中详细说明。

## 支持的平台

Cowboy在Linux，FreeBSD，Windows和OSX上经过测试和支持。

Cowboy也可以在其他平台上工作，但我们不保证体验安全顺畅。 建议您在部署到其他平台之前执行必要的测试和安全审核。

Cowboy是为Erlang/OTP 19.0及以上版本开发的。


## License

Cowboy 使用ISC许可证。

```
Copyright (c) 2011-2017, Loïc Hoguin <essen@ninenines.eu>

Permission to use, copy, modify, and/or distribute this software for any
purpose with or without fee is hereby granted, provided that the above
copyright notice and this permission notice appear in all copies.

THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES
WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF
MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR
ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES
WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN
ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF
OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.
```

## Versioning

Cowboy使用Semantic Versioning 2.0.0](https://semver.org/)。

## 约定

在HTTP协议中，方法名称区分大小写。 所有标准方法名称都是大写的。

标题名称不区分大小写。 使用HTTP/1.1时，Cowboy会将所有请求标头名称转换为小写。 HTTP/2要求客户端以小写形式发送它们。 任何其他标题名称都应该以小写形式提供，包括查询有关请求的信息或发送响应时。

这同样适用于任何其他不区分大小写的值。