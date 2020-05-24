# 怎么读ES规范

先把前4个clause过一遍，然后看看目录，有个印象。

## 例1 `==`

`==`语法解释在12.11 Expressions -- Equality Operators，这个Static Semantics没什么好看的，
看Runtime Semantics (RS)，找到`==`的部分，

> 1. Let lref be the result of evaluating EqualityExpression.
> 2. Let lval be ? GetValue(lref).
> 3. Let rref be the result of evaluating RelationalExpression.
> 4. Let rval be ? GetValue(rref).
> 5. Return the result of performing ***Abstract Equality Comparison*** rval == lval.

其中的evaluating和GetValue不懂就先跳过（有时间看看Reference），总之就是lval是左边表达式的值，rval是右边的值，
所以这里只用看最后一句的Abstract Equality Comparison，直接照着链接点进去就行了，跳到7.2.15。

> The comparison x == y, where x and y are values, produces true or false. Such a comparison is performed as follows:
>
> 1. If Type(x) is the same as Type(y), then
>    1. Return the result of performing Strict Equality Comparison x === y.
> 2. If x is null and y is undefined, return true.
> 3. If x is undefined and y is null, return true.
> 4. ...

现在应该明朗了，

1. 同类型的话，返回x === y的结果
2. null == undefined
3. undefined == null
4. ...后面ToNumber, ToPrimitive一样的点链接过去


## 例2 `f()`

调用函数的语法在 12.3.6，看看RS，可以通过链接依次查看`EvaluateCall` - `Call`

> ... It is used to call the [[Call]] internal method of a function object. F is the function object,  ...
>
> 1. If argumentsList is not present, set argumentsList to a new empty List.
> 2. If IsCallable(F) is false, throw a TypeError exception.
> 3. Return ? F.[[Call]](V, argumentsList).

到这里发现`[[Call]]`没有链接了，这是因为根据F的不同F.[[Call]]的语义也不同。

这里知道F是个function object，点击链接到6.1.7.2先看看。发现[[Call]]和[[Construct]]在第9章（[[XXX]]都在这）

> 9.2 ECMAScript Function Objects
> 9.3 Built-in Function Objects
> 9.4.1 Bound Function Exotic Objects

字面意思，根据实际情况选择，之后就继续链接了。

