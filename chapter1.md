# 优化杀手

_**原文地址：**_[_**https://github.com/petkaantonov/bluebird/wiki/Optimization-killers**_](https://github.com/petkaantonov/bluebird/wiki/Optimization-killers)

## 引言

这篇文档包含了如何避免使代码性能远低于预期的建议. 尤其是一些会导致 V8 \(牵涉到 Node.js, Opera, Chromium 等\) 无法优化相关函数的问题.

[vhf](/vhf "https://github.com/vhf")\(nodejs 作者\)也有一个类似的项目，尝试去列举出所有v8 Crankshaft引擎的杀手们：[V8 Bailout Reasons](/V8 Bailout Reasons "https://github.com/vhf/v8-bailout-reasons")

### V8背景知识

在 V8 中并没有解释器, 但却有两个不同的编译器: 通用编译器和优化编译器. 这意味着你的 JavaScript 代码总是会被编译为机器码后直接运行. 这样一定很快咯? 并不是. 仅仅是编译为本地代码并不能明显提高性能. 它只是消除了解释器的开销, 但如果未被优化, 代码依旧很慢.

举个例子, 使用通用编译器,`a + b`会变成这个样子:

```
mov eax, a
mov ebx, b
call RuntimeAdd
```

换言之它仅仅是调用了运行时的函数. 如果a和b是整数（常量？）, 那可以像这样:

```
mov eax, a
mov ebx, b
add eax, ebx
```

（应该是指常量折叠吧，可以看看[这篇文章](/这篇文章 "https://zhuanlan.zhihu.com/p/25067384&quot;\)）\]\(https://zhuanlan.zhihu.com/p/25067384&quot;")\)

通常来说, 通用编译器得到的是第一种结果, 而优化编译器则会得到第二种结果. 使用优化编译器编译的代码可以很容易比通用编译器编译的代码快上 100 倍. 但这里有个坑, 并非所有的 JavaScript 代码都能被优化. 在 JavaScript 中有很多种写法, 包括具备语义的, 都不能被优化编译器编译 \(回落到通用编译器[^1]\).

记下一些会导致整个函数无法使用优化编译器的用法很重要. 一次代码优化的是一整个函数, 优化过程中并不会关心其他代码做了什么 \(除非代码在已经被优化的函数中\).

这个指南会涵盖多数会导致整个函数掉进 “反优化火狱” 的例子. 由于编译器一直在不断更新, 未来当它能够识别下面的一些情况时, 这里提到的处理方法可能也就不必要了.

## 目录

* ### [工具](#工具 "工具")
* ### [不支持的语法](#不支持的语法 "不支持的语法")
* ### [argumens管理](#argumens管理 "argumens管理")
* ### [switch case](#switch-case "switch case")
* ### [for in](#for-in "for in")
* ### [具有深度逻辑或不清晰退出条件的死循环](#具有深度逻辑或不清晰退出条件的死循环 "具有深度逻辑或不清晰退出条件的死循环")

## 工具

你可以通过添加一些 V8 标记来使用 Node.js 验证不同的用法如何影响优化结果. 通常可以写一个包含了特定用法的函数, 使用所有可能的参数类型去调用它, 再使用 V8 的内部函数去优化和审查：

test.js:

```js
//Function that contains the pattern to be inspected (using an `eval` statement)
function exampleFunction() {
    return 3;
    eval('');
}

function printStatus(fn) {
    switch(%GetOptimizationStatus(fn)) {
        case 1: console.log("Function is optimized"); break;
        case 2: console.log("Function is not optimized"); break;
        case 3: console.log("Function is always optimized"); break;
        case 4: console.log("Function is never optimized"); break;
        case 6: console.log("Function is maybe deoptimized"); break;
        case 7: console.log("Function is optimized by TurboFan"); break;
        default: console.log("Unknown optimization status"); break;
    }
}

//Fill type-info
exampleFunction();
// 2 calls are needed to go from uninitialized -> pre-monomorphic -> monomorphic
exampleFunction();

%OptimizeFunctionOnNextCall(exampleFunction);
//The next call
exampleFunction();

//Check
printStatus(exampleFunction);
```

运行一下：

```
$ node --trace_opt --trace_deopt --allow-natives-syntax test.js
(v0.12.7) Function is not optimized
(v4.0.0) Function is optimized by TurboFan
```

[TurboFan](/TurboFan "https://codereview.chromium.org/1962103003")

作为是否被优化的对比, 注释掉`eval`语句再来一次:

```
$ node --trace_opt --trace_deopt --allow-natives-syntax test.js
[optimizing 000003FFCBF74231 <JS Function exampleFunction (SharedFunctionInfo 00000000FE1389E1)> - took 0.345, 0.042, 0.010 ms]
Function is optimized
```

学会使用这个工具是非常重要和必要的，不然你怎么去验证？

## 不支持的语法

优化编译器不支持一些特定的语句, 使用这些语法会使包含它的函数无法得到优化.

有一点请注意, 即使这些语句无法到达或者不会被执行, 它们也会使相关函数无法被优化.

比如这样做是没用的:

```js
if (DEVELOPMENT) {
    debugger;
}
```

上面的代码会导致包含它的整个函数不被优化, 即使从来不会执行到 debugger 语句.

目前不会被优化的有:

* 包含含有`__proto__`或者`get/set`声明的对象字面量的函数

即，如果有以下情况，整个函数将无法被优化：

```js
function containsObjectLiteralWithProto() {
    return { __proto__: 3 };
}
```

```js
function containsObjectLiteralWithGetter() {
    return {
        get prop() {
            return 3;
        }
    };
}
```

```js
function containsObjectLiteralWithSetter() {
    return {
        set prop(val) {
            this.val = val;
        }
    };
}
```

可能永远不会被优化的有:

* 包含`debugger`语句的函数
* 包含字面调用`eval()`的函数
* 包含`with`语句的函数

提一下直接使用`eval`和`with`的情况, 因为它们会造成相关嵌套的函数作用域变为动态的. 这样一来则有可能也影响其他很多函数, 因为这种情况下无法从词法上判断相关变量的有效范围.

### 攻略

之前提到过的一些语句在生产环境中是无法避免的, 比如`try...finally`和`try...catch.`为了是代价最小, 它们必须被隔离到一个最小化的函数, 以保证主要的代码不受影响.

```js
var errorObject = { value: null };
function tryCatch(fn, ctx, args) {
    try {
        return fn.apply(ctx, args);
    } catch(e) {
        errorObject.value = e;
        return errorObject;
    }
}

var result = tryCatch(mightThrow, void 0, [1,2,3]);
// 不带歧义地判断是否调用抛出了异常 (或其他值)
if(result === errorObject) {
    var error = errorObject.value;
} else {
    // 结果是返回值
}
```

try catch 现在已经可以优化了，但这个攻略依然还是有意义的，而且浏览器不一定支持了新版本的V8

## argumens管理

有不少使用`arguments`的方式会导致相关函数无法被优化. 所以在使用`arguments`的时候需要非常留意.

### 给一个已经定义的参数重新赋值, 并且在相关语句主体中引用arguments \(仅限非严格模式\). 典型的例子:

```js
function defaultArgsReassign(a, b) {
     if (arguments.length < 2) b = 5;
}
```

**攻略**是将值拷贝到一个新变量上

```js
function reAssignParam(a, b_) {
    var b = b_;
    // 与 b_ 不同, b 可以安全地被重新赋值
    if (arguments.length < 2) b = 5;
}
```

如果只是参数默认值要用到arguments，还不如直接判断undefined，如下：

```js
function reAssignParam(a, b) {
    if (b === void 0) b = 5;
}
```

但是如果后面的代码有得引入arguments，那么这样赋值是没意义的。

**攻略2：**函数或者文件开启严格模式

### 泄漏argumens：

```js
function leaksArguments1() {
    return arguments;
}
```

```js
function leaksArguments2() {
    var args = [].slice.call(arguments);
}
```

```js
function leaksArguments3() {
    var a = arguments;
    return function() {
        return a;
    };
}
```

坚决不要泄漏arguments

**攻略**是拷贝到一个数组里面再泄漏出来

```
function doesntLeakArguments() {
                    // .length 只是一个整数, 它不会泄露
                    // arguments 对象本身
    var args = new Array(arguments.length);
    for(var i = 0; i < args.length; ++i) {
                // i 始终是 arguments 对象的有效索引
        args[i] = arguments[i];
    }
    return args;
}
```

```
function anotherNotLeakingExample() {
    var i = arguments.length;
    var args = [];
    while (i--) args[i] = arguments[i];
    return args
}
```

写一堆代码很让人恼火, 所以分析是否值得这么做是值得的. 接下来更多的优化总是会带来更多的代码, 而更多的代码又意味着语义上更显而易见的退化.

然而如果你有一个 build 步骤, 这其实可以用一个不需要 source map 的宏来实现, 同时保证源代码是有效的 JavaScript 代码.、

```
function doesntLeakArguments() {
    INLINE_SLICE(args, arguments);
    return args;
}
```

上面的技巧就用到了 Bluebird 中, 在 build 后会被扩充为下面这样:

```
function doesntLeakArguments() {
    var $_len = arguments.length;
    var args = new Array($_len); 
    for(var $_i = 0; $_i < $_len; ++$_i) {
        args[$_i] = arguments[$_i];
    }
    return args;
}
```

### 对arguments赋值

在非严格模式下, 这其实是可能的:

```
function assignToArguments() {
    arguments = 3;
    return arguments;
}
```

**攻略是**没必要写这么蠢的代码. 说来在严格模式下, 它也会直接抛出异常.

### 怎样安全地使用arguments？

仅使用：

* `arguments.length`
* `argumens[i]` **i必须是arguments的有效整数索引，且不能超出边界**
* 除了`.length`和`[i]`, 永远不要直接使用`arguments`

* 严格地说，`fn.apply(y, arguments)`可以用，其他都不行，比如`.slice`。`Function#apply`太特殊了

* 但是有一种情况用`#apply`是不安全的，就是这个functions fn被添加了新的属性，或者被绑定函数（`Function#bind`）搞了以生成隐藏类

只要你用以上方法去操作arguments，就可以放心大胆地使用arguments了

## switch case

一个 switch…case 语句目前可以有最多 128 个 case 从句, 如果超过了这个数量, 包含这个 switch 语句的函数就无法被优化.

```js
function over128Cases(c) {
    switch(c) {
        case 1: break;
        case 2: break;
        case 3: break;
        ...
        case 128: break;
        case 129: break;
    }
}
```

所以请保证 switch 语句的 case 从句不超过 128 个, 可以使用函数数组或者 if…else 代替.

## for in

for…in 语句在一些情况下可能导致包含它的函数无法被优化.

下面是for-in不快的几个case

### 键不是局部变量

```js
function nonLocalKey1() {
    var obj = {}
    for(var key in obj);
    return function() {
        return key;
    };
}
```

```js
var key;
function nonLocalKey2() {
    var obj = {}
    for(key in obj);
}
```

key既不能来自上层作用域，也不能让子作用域引用，他必须是局部变量

### 被枚举的对象不是 “简单可枚举的”

#### 处于 “哈希表模式” 的对象 \(即 “普通化的对象”, “字典模式” – 以哈希表为数据辅助结构的对象\) 不是简单的可枚举对象

```js
function hashTableIteration() {
    var hashTable = {"-": 3};
    for(var key in hashTable);
}
```

如果你 \(在构造函数外\) 动态地添加太多属性到一个对象, 删除属性, 使用不是合法标识符 \(identifier\) 的属性名称, 这个对象就会变为哈希表模式. 换言之, 如果你把一个对象当做哈希表来使用, 它就会转变为一个哈希表. 不要再 for…in 中使用这样的对象. 判断一个对象是否为哈希表模式, 可以在开启 Node.js 的

`--allow-natives-syntax`

选项时调用

`console.log(%HasFastProperties(obj))。`

#### 对象的原型链中有可枚举的属性

```
Object.prototype.fn = function() {};
```

添加上面的代码会使所有的对象 \(除了`Object.create(null)`创建的对象\) 的原型链中都存在一个可枚举的属性. 由此任何包含 for…in 语句的函数都无法得到优化 \(除非仅枚举`Object.create(null)`创建的对象\).

你可以通过`Object.defineProperty`来创建不可枚举的属性 \(不推荐运行时调用, 但是高效地定义一些静态的东西, 比如原型属性, 还是可以的\).

### 对象包含可枚举的数组索引

一个属性是否是数组索引是在[ECMAScript 规范](http://www.ecma-international.org/ecma-262/5.1/#sec-15.4)中定义的

> A property name P \(in the form of a String value\) is an array index if and only if ToString\(ToUint32\(P\)\) is equal to P and ToUint32\(P\) is not equal to 232−1. A property whose property name is an array index is also called an element

通常来说这些对象是数组, 但普通的对象也可以有数组索引:`normalObj[0] = value;`

```js
function iteratesOverArray() {
    var arr = [1, 2, 3];
    for (var index in arr) {

    }
}
```

所以使用 for…in 遍历数组不仅比 for 循环慢, 还会导致包含它的整个函数无法被优化.

---

如果传递一个非简单的可枚举对象到 for…in, 会导致整个函数无法被优化.

**攻略：**总是使用`Object.keys`再使用 for 循环遍历数组. 如果的确需要原型链上的所有属性, 创建一个单独的辅助函数.

```js
function inheritedKeys(obj) {
    var ret = [];
    for(var key in obj) {
        ret.push(key);
    }
    return ret;
}
```

## 具有深度逻辑或不清晰退出条件的死循环

写代码的时候, 有时会知道自己需要一个循环, 但不清楚循环内的代码会写成什么样子. 所以你放了一个`while (true) {`或者`for (;;) {`, 之后再在一定条件下中断循环接续之后的代码, 最后忘了这么一件事. 重构的时间到了, 你发现这个函数很慢, 或者发现一个反优化[^2]的情况 – 可能它就是罪魁.

将循环的退出条件重构到循环自己的条件部分可能并不容易. 如果代码的退出条件是结尾 if 语句的一部分, 并且代码至少会执行一次, 那可以重构为`do { } while ();`循环. 如果退出条件在循环开头, 把它放进循环本身的条件部分. 如果退出条件在中间, 你可以尝试 “滚动” 代码: 每每从开头移动一部分代码到末尾,也可以复制一份到循环开始之前. 一旦退出条件可以放置在循环的条件部分, 或者至少是一个比较浅的逻辑判断, 这个循环应该就不会被反优化了.

[^1]: bail out

