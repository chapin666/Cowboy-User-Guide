# Constraints

约束是应用于用户输入的验证和转换函数。

它们在Cowboy的各个地方使用，包括路由器和cowboy_req匹配功能。

## 语法

约束作为字段列表提供。对于列表中的每个字段，可以应用特定约束，如果缺少字段，则应用默认值。

字段可以采用原子字段的形式，具有约束{field，Constraints}的元组或具有约束的元组和默认值{field，Constraints，Default}。字段表单表示该字段是必填字段。

请注意，当与路由器一起使用时，只有第二种形式有意义，因为它不使用默认值，并且始终定义字段。

每个字段的约束作为要应用的原子或函数的有序列表提供。内置约束作为原子提供，而自定义约束作为funs提供。

当提供多个约束时，它们按给定的顺序应用。如果该值已被约束修改，则下一个值将接收新值。

例如，以下约束将首先验证字段my_value并将其转换为整数，然后检查整数是否为正数：
```
PositiveFun = fun
    (_, V) when V > 0 ->
        {ok, V};
    (_, _) ->
        {error, not_positive}
end,
{my_value, [int, PositiveFun]}.
```

我们忽略了此代码段中的第一个有趣的参数。 我们不应该。 我们将简单地了解本章后面的内容。

当只有一个约束时，可以直接提供它而不将其包装到列表中：
```
{my_value, int}
```

## 内置约束

内置约束被指定为原子：

| Constraint|Description|
|-----------------|------------------------|
| int | 转换二进制成int类型 |
| nonempty | 确认二进制值不为空|

## 自定义约束

自定义约束被指定为有趣的。 这个乐趣有两个论点。 第一个参数表示要执行的操作，第二个参数表示值。 值是什么以及必须返回什么取决于操作。

Cowboy目前定义了三个操作。 用于验证和转换用户输入的操作是正向操作。
```
int(forward, Value) ->
    try
        {ok, binary_to_integer(Value)}
    catch _:_ ->
        {error, not_an_integer}
    end;
```
即使未通过约束转换，也必须返回该值。

反向操作则相反：它采用转换后的值并将其更改回用户输入的值。
```
int(reverse, Value) ->
	try
		{ok, integer_to_binary(Value)}
	catch _:_ ->
		{error, not_an_integer}
	end;
```

最后，format_error操作接受任何其他操作返回的错误，并返回格式化的人类可读错误消息。
```
int(format_error, {not_an_integer, Value}) ->
	io_lib:format("The value ~p is not an integer.", [Value]).
```
请注意，对于这种情况，您将获得错误以及为产生此错误的约束赋予的值。

Cowboy不会捕获来自约束函数的异常。 它们应该写成不排除任何例外。