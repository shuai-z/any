# block

`#sec-blockdeclarationinstantiation`

进入block会创建新的作用域，用于绑定声明在这个block内部的`let, const, function, class`。

block内部的var绑定在其外面的某个作用域。

# function declaration

在脚本或函数顶层是，函数声明会被提升（hoist） (`#sec-block-static-semantics-toplevelvarscopeddeclarations`)

"提升"的作用类似于把函数声明当作 `var f = ...`，但是它会初始化为这个函数（closure）而不是undefined，所以可以在函数声明*之前*使用。

```js
console.log(f); // 打印f

//...

function f() {}
```

# function declaration in block

因为函数声明是个declaration，也出现在LexicallyScopedDeclarations，所以在block内部会为它创建绑定。

但是，这样的话，语言的使用者就有点疑问了：此时还要不要“提升”成`var`？

```js
{
  function f() {}
}
f();
```

历史原因，这部分放到了附录B.3.3，由引擎自己决定。

下面举几个例子看看“提升”之后需要注意的地方，

a) 可以当成var的前提条件是：不会引起新的语法错误，并且没有同名的参数

```js
function f(g) {
  let h = 1;

  {
    function g() {}
    function h() {}
  }
  
  console.log(g, h);
}
```

b) block之前不能调用，因为还不能确定

```js
f() // 错误，此时f是undefined

if (x) {
  function f() {}
} else {
  function f() {}
}

f() // ok
```

c) 另外，因为block里绑定的这个是Lexical的声明，类似let，所以也不能有重名

```js
{
  // 错误，f重名
  var f;
  function f() {}
}

// 并且一样是在一开始就初始化好的
{
  console.log(f); // ok
  function f() {}
}
```

总结起来就是，当函数声明在block里面的时候，它既当成let绑定一次，又(可能)当成var在外面绑定一次。也就是说...

```js
{
  function f() {}
  f = 2; // 改成数字2
  console.log(f); // 确认是2
}
f() // 还能调用...
```

下面简单解释一下为什么

```js
// varEnv创建f绑定，并初始化为undefined

{ // 创建newEnv
  // newEnv创建f绑定，并初始化为函数closure
  // 注意此时varEnv里的f绑定依然是undefined

  f(); // ok，因为newEnv里的f已经是个函数

  function f() {} // 此时将newEnv里f的值赋值给varEnv的f

  f = 2; // 更改的是只是newEnv里的f的值
}

// varEnv的f绑定是函数，所以可以调用
f();
```

最后看个例子，打印什么？

```js
{
  console.log(f);
  f = 1;
  function f() {}
}
console.log(f);
```

以上写的这些注意和解释，并不是想说“如何正确的使用”，而是 ***不要在block里声明函数***。

