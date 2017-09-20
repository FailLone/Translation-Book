# Bluebird中的toFastProperties方法是怎么给对象提速的
_**原文地址：**_[_**https://stackoverflow.com/questions/24987896/how-does-bluebirds-util-tofastproperties-function-make-an-objects-properties#**_](https://stackoverflow.com/questions/24987896/how-does-bluebirds-util-tofastproperties-function-make-an-objects-properties#)

在`Bluebird`的[`util.js`](https://github.com/petkaantonov/bluebird/blob/master/src/util.js#L201)文件里有如下方法：
```js
function toFastProperties(obj) {
    /*jshint -W027*/
    function f() {}
    f.prototype = obj;
    ASSERT("%HasFastProperties", true, obj);
    return f;
    eval(obj);
}
```
不知道为啥，在return后面居然还写了一些代码。
而且这代码明显是不可达的，作者也特地声明禁用了`jshint`的警。

>Unreachable 'eval' after 'return'. (W027)

这个方法到底是干嘛的？他真的能使得object更快吗？

***

2017年已经更新啦，对于新来的读者，以下是现在的版本，可在`Node7(4+)`中运行。
```js
function enforceFastProperties(o) {
    function Sub() {}
    Sub.prototype = o;
    var receiver = new Sub(); // create an instance
    function ic() { return typeof receiver.foo; } // perform access
    ic(); 
    ic();
    return o;
    eval("o" + o); // ensure no dead code elimination
}
```
## 除去一两个小的优化点，以下所有的都仍是有效的
我们首先来讨论这个方法在干嘛，为什么会变快，他是如何工作的。
## 他在干嘛
V8引擎有两种表达对象的方法：

* **字典模式**- 和[hash map](https://en.wikipedia.org/wiki/Hash_table)一致，使用名值对存储对象
* **快速模式**- 和[structs](https://en.m.wikipedia.org/wiki/Struct_\(C_programming_language\))类似，特点是属性存取时不需要计算

[这里](https://jsperf.com/test-dictionary-mode)验证了这两种模式的性能差别。这里是使用`delete`来强制对象降级为字典模式

引擎会在一切可能的情况下使用快速模式，即使是大量属性存取在操作的情况。但是有些情况不得不降级成字典模式。在字典模式下会出现极大的性能损失，所以一般来说，我们都会中意快速模式。

这里有一些hack就是尝试将对象由字典模式转为快速模式

* Bluebird的Petka自己[提到的](Optimization%20killers.md)
* Vyacheslav Egorov的[ppt](http://mrale.ph/s3/nodecamp.eu/#54)
* [一些问题和已接受的回答](https://stackoverflow.com/questions/23455678/pros-and-cons-of-dictionary-mode)也提到了
* [略过时的文章](http://jayconrod.com/posts/52/a-tour-of-v8-object-representation)，但是依然值得一读，你会从中学到对象在v8中是怎么存储的
## 为什么会变快
JavaScript原型链就很典型地在各个实例间共享了方法，并且很少会大幅度的变化。鉴于此，必须保持他们在快速模式下，才能很好的在每次函数调用时避免性能损耗。

所以，v8才会把函数的`prototype`属性对象搞成快速模式，因为函数被作为构造函数调用时创建的所有对象都将共享。这个优化是很聪明和有效的。
## 他是怎么工作的
我们首先通过代码和注释去讲解每一行是干嘛的：
```js
function toFastProperties(obj) {
    /*jshint -W027*/ // 干掉不可达代码的报错
    function f() {} // 申明一个新函数
    f.prototype = obj; // 将obj指定为它的prototype，以触发优化
    // 保持这个优化是有效的，以免今后这段代码失效以至优化失败
    ASSERT("%HasFastProperties", true, obj); // 需要"native syntax"
    return f; // 返回
    eval(obj); // 避免函数被优化，包括死代码清除或进一步优化。这句代码不可达，但是即便是不可达代码，只要使用了eval依然可以阻止v8优化函数

}
```

没有必要去自己去找这样做v8到底会不会优化，我们只用看[v8的单元测试](https://github.com/v8/v8/blob/d52280b1a7a867ffb350c4f193cf8692861855dd/test/mjsunit/fast-prototype.js)就够了:
```js
// Adding this many properties makes it slow.
assertFalse(%HasFastProperties(proto));
DoProtoMagic(proto, set__proto__);
// Making it a prototype makes it fast again.
assertTrue(%HasFastProperties(proto));
```

跑一跑这段单元测试，你会发现的确v8的确优化了，我们接下来看看，这是怎么做的

首先看`objects.cc`，有以下函数(L9925)：
```c++
void JSObject::OptimizeAsPrototype(Handle<JSObject> object) {
  if (object->IsGlobalObject()) return;

  // Make sure prototypes are fast objects and their maps have the bit set
  // so they remain fast.
  if (!object->HasFastProperties()) {
    MigrateSlowToFast(object, 0);
  }
}
```
`JSObejct::MigrateSlowToFast`接受一个字典模式的对象，然后将其转换为V8快速模式。读一读v8对象内部的秘密是值得的，但我们这儿不进一步展开了。但我还是要强烈推荐你阅读[这篇文章](https://github.com/v8/v8/blob/3235f3f8b5930de07a240f61386f21d55040dbf8/src/objects.cc#L4617-L4751)，从中你可以很好地学习v8对象。

接着看`objects.cc`中的`SetPrototype`方法，我们可以看到12231行，`OptimizeAsPrototype`被调用了：
```c++
if (value->IsJSObject()) {
    JSObject::OptimizeAsPrototype(Handle<JSObject>::cast(value));
}
```
`SetPrototype`又被`FuntionSetPrototype`调用，后者就是我们的`.prototype = `。

使用`__proto__ = `和`.setPrototypeOf`也有同样的效果，但是他们是ES6的函数，且Bluebird支持自Netscape 7以来的所有浏览器，所以用他们来优化代码是不可能的。举个例子，看`.setPrototypeOf`实现：
```c++
// ES6 section 19.1.2.19.
function ObjectSetPrototypeOf(obj, proto) {
  CHECK_OBJECT_COERCIBLE(obj, "Object.setPrototypeOf");

  if (proto !== null && !IS_SPEC_OBJECT(proto)) {
    throw MakeTypeError("proto_object_or_null", [proto]);
  }

  if (IS_SPEC_OBJECT(obj)) {
    %SetPrototype(obj, proto); // MAKE IT FAST
  }

  return obj;
}
```
该方法直接在`object`上：
```c++
InstallFunctions($Object, DONT_ENUM, $Array(
...
"setPrototypeOf", ObjectSetPrototypeOf,
...
));
```
所以还得用Petka的实现方式。

#### 免责声明

记住这些是所有的实现细节。Petka这种逼是那种优化怪咖。请铭记，97%的情况下，过早的优化都是万恶之源。Bluebird做了很多很基础的这种工作，才能从这些优化hack中收获良多，尽可能快地响应callback不是一件容易事啊。所以，你基本会很少用到这些代码。

#### 顺便说一下
更新是因为:
* 现在v8必须实例化构造函数才会优化
* 以前的代码Node7+无效

且，现在又更新，目前的代码是：
```js
function toFastProperties(obj) {
    /*jshint -W027,-W055,-W031*/
    function FakeConstructor() {}
    FakeConstructor.prototype = obj;
    var l = 8;
    while (l--) new FakeConstructor();
    ASSERT("%HasFastProperties", true, obj);
    return obj;
    // Prevent the function from being optimized through dead code elimination
    // or further optimizations. This code is never reached but even using eval
    // in unreachable code causes v8 to not optimize functions.
    eval(obj);
}
```