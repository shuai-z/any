> NOTE:
>
> 1. `#sec-xxx` 表示对应spec的具体章节*https://tc39.es/ecma262/#sec-xxx*
> 2. 不讨论不同引擎不同版本的实现差异

# 1. Environment Records

`#sec-environment-records`

“作用域”？

> *environment record*（为了方便，下面直接写为*env*）记录了变量名与值的绑定关系，
>
> *env*还有一个[[OuterEnv]]指向*outer env*，关系...类似proto


```js
{
  let a = 1;
  {
    let b = 2;
  }
}
```

执行这段代码会先后创建两个*env*，

*env1*: （*env1*.[[OuterEnv]] = *GlobalEnv*)

| name | value |
|---|---|
| a | 1 |

*env2*: (*env2*.[[OuterEnv]] = *env1*)

| name | value |
|---|---|
| b | 2 |

里面的block可以访问到外面的a是因为会递归查[[OuterEnv]]

> `#sec-getidentifierreference`
>
> 当前*env*里没有这个记录的绑定关系时到[[OuterEnv]]里找，直到*env*为null

实际上绑定关系不止name和value，还有是否initialized，能否被delete。

| name | value | is initialized | can be deleted |
|---|---|---|---|


## hierarchy

env分为以下具体类别：declarative env，object env, global env。
其中declarative env还可分为function env和module env。

- *object env*有一个binding object，查找时可以在这个对象里找，所以with里可以直接用`prop`代替`o.prop`；
- 执行函数时会创建*function env*，它比*declarative env*多一些属性，比如`this`指向的值；

*global env*有[[ObjectRecord]]（类型object env）和[[DeclarativeRecord]]（类型declarative env），
查变量时会先查[[DeclarativeRecord]]再查[[ObjectRecord]]，
详细的解释在`#sec-global-environment-records`

```js
var a = 1; // 绑定关系在ObjectRecord，所以window.a === a
let b = 1; // 绑定关系在DeclarativeRecord
NaN; parseInt; // ... 这些都是在window上的
```

## 相关错误

判断是否会抛出错误以及会抛出什么错误的逻辑是一样的，

1. 是否存在绑定？
2. 绑定是否已初始化？


# 2. let/var

首先let跟var的最显著区别是，var是个statement，let是个declaration（无提升），这也是导致结果不同的原因。


### 1. script

`#sec-globaldeclarationinstantiation`

```js
var a = a;
let b = b;
```

执行这段代码体之前会先有一个初始化的阶段(`#sec-globaldeclarationinstantiation`)，暂时简化为

```js
let env = GlobalEnv // 全局环境
env.createMutableBinding('b', false) // 在env.[[DeclarativeRecord]]中创建绑定，注意并没有初始化
env.createGlobalVarBinding('a', false) // 在env.[[ObjectRecord]]中创建绑定，并初始化为undefined
```

> 一个没有被初始化的绑定，取值的时候会抛出ReferenceError:...未初始化。（`getbindingvalue`）

```js
var a = a; // a的绑定存在，且已初始化为undefined，所以就是把undefined赋值给a
let b = b; // b的绑定存在，但未初始化，所以取b的值会抛出ReferenceError: Cannot access 'b' before initialization
```


### 2. function

`#sec-functiondeclarationinstantiation`

下面这个函数执行时会抛出ReferenceError: Cannot access 'b' before initialization

```js
function f() {
  var a = a;
  let b = b;
}
```

后面会单独写一下函数的env。比如这个函数在strict模式下a跟b在同一个env，不在strict模式就是两个env。


### 3. block

`#sec-blockdeclarationinstantiation`

```js
{
  var a = 1;
  let b = 2;
}
```

a在global env里，在运行第一行之前就已经绑定并初始化为undefined；

当执行到这个block的时候，会创建一个新的declarative env，然后绑定b但是没初始化，
这个新的env就是当前运行上下文的env(LexicalEnvironment)，
执行到a=1时会把global env的a绑定初始化，
执行到b=2时把这个新env的b绑定初始化，
block的语句都执行完毕后恢复原来的env，这个刚创建的env就不再需要了。

# 3. function declaration

函数声明会被提升（hoist），这其中的语法解释在 `#sec-block-static-semantics-toplevelvarscopeddeclarations`

```js
// 类似var f = ...，区别在于f不是初始化为undefined而是这个函数（closure）
function f() {}
```

另外一个值得注意的是function declaration是一个declaration，
它除了出现在上面说的TopLevelLexicallyScopedDeclarations，
同时也出现在LexicallyScopedDeclarations。

...所以，下面的代码两个env里都有f的绑定关系

```js
{
  function f() {}
  console.log(f);
}
console.log(f);
```

也就是说，你可以改变其中一个而不影响另一个

```js
{
  function f() {}
  f = 2; // 改为数字2
}
// 这里的f还是函数
f()
```

