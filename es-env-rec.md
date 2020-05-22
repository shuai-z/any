**NOTE:**

1. `#sec-xxx` 表示对应spec的具体章节*https://tc39.es/ecma262/#sec-xxx*
2. 引擎的实现并不完全是这样的

# 1. Environment Records

`#sec-environment-records`

*environment record*（或者叫“作用域”，下面直接写为*env*）记录了变量名与值的绑定关系。比如`let a = 1`在当前环境(lexical environment)里创建`a`的绑定。

只是记录自己这层的绑定关系还不够，比如函数（或block）里面要能使用外面的变量，所以*env*还要有一个[[OuterEnv]]指向*outer env*。

> `#sec-getidentifierreference`
>
> 当前*env*里没有绑定时，递归到[[OuterEnv]]里找，直到*env*为null

绑定关系记录的除了name和value外，还有是否初始化，能否被删除，以及是否可更改。

| name | value | is initialized | can be deleted |
|---|---|---|---|


## hierarchy

* DeclarativeEnv
  * FunctionEnv，比DeclarativeEnv多一些属性，比如`this`
  * ModuleEnv
* ObjectEnv，有一个binding object，查找时可以在这个对象里找，所以with里可以直接用`prop`代替`o.prop`
* GlobalEnv

*GlobalEnv*有[[ObjectRecord]]（类型ObjectEnv）和[[DeclarativeRecord]]（类型DeclarativeEnv），
查变量时会先查[[DeclarativeRecord]]再查[[ObjectRecord]]。(`#sec-global-environment-records`)

```js
var a = 1; // 绑定关系在ObjectRecord，所以window.a === a
let b = 1; // 绑定关系在DeclarativeRecord
```

## 相关错误

1. 是否存在绑定？对应`RefrenceError: xxx is not defined`。
2. 绑定是否已初始化？对应`ReferenceError: cannot access xxx before initialization`。

```js
var a = a; // var会初始化为undefined，所以没问题
let b = b; // let没初始化
```

似乎规范里并没有记录一个绑定是var来的还是let来的，但很多时候又是要区别对待的...

