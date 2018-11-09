---
date: 2018-11-09T18:38:30+08:00
tags: 
- javascript
- 原型链
- prototype
- proto
categories:
- 周记
title: Joseph的奇妙原型链解读
---


正文
---
_Joe按，用自己的话来聊一聊这个问题，希望能让不写js的人也看得明白。_

### 对象、原型对象、原型链

第一个概念，当你通过`.`操作符访问一个JS对象的属性时，先从自身属性（Object.hasOwnProperty能访问的那层）开始找，如果找不到该属性，再会通过对象的原型对象去找这个属性，如果还没有，就去原型对象的原型对象上找，直到原型对象没有原型对象为止，都没找到，返回undefined。这个原型对象组合起来的链式结构，称之为原型链。JS中所有的继承关系都由这个原型链来实现。

注意我这里用的都是中文词，而没有使用`prototype`, `constructor`, `__proto__`之类的词语，就是为了暂时抛开这些容易混淆的概念。目前的关键词只有
  1. 对象
  2. 原型对象
  3. 原型对象组成的原型链
  
下面我们一个一个把概念引进来。

### constructor 构造器

每个函数(function)都可以当作构造器(constructor)使用，函数和构造器的区别在于，函数就是用来执行的，构造器是用来创建实例的，它们的表现形式都是一个function，不同的叫法表达了它不同的用处。

作为构造器使用的时候，需要`new`关键词来发动。

每当对一个函数使用`new`关键字，可以理解为js引擎会在你代码开始执行之前，创建一个`new Object`并指向`this`, 然后执行函数里的代码，执行完之后加一句`return this;`。所以如果你在这句话执行前自己return了一些东西，就能拦截构造函数把新对象返回了哟。
皮一下验证我们的理解对不对。
```
var obj = {'hoho': 'new谁都是我啦'}

function Factory() {
  return obj
}

var foo = new Factory

var bar = new Factory

foo.hoho
// "new谁都是我啦"
foo === obj
/// true
foo === bar
// true
```

遥相呼应一下`this`指向的问题，由于JS语言的动态性，判断this指向的时候往往先要分清编译时和执行时的概念。

构造器(constructor)对应的是编译时(只下定义不执行)，实例instance对应的是执行时(执行的时候才会产生构造器的实例嘛)。

this本身表达的就是一个执行时的调用方。这里有个一刀切的判断就是如果看到调用时`new`关键词，函数体内的this永远指向一个你new出来的临时新对象。如果把返回值赋值给某个变量，那么就等于保存了它的引用，如果不保存，它就会被js的垃圾回收(garbage collection)机制回收走，成为一个名副其实的临时对象。

展开得有点多，但其实是为了更丰富地描述构造器(constructor)和实例(instance)的区别。

回归正题，我们上面提及的对象、原型对象，都是实例(instance)，原型链是一个串着各种instance的链。

### `__proto__` 原型对象

JS的世界里一切皆对象，一切对象都有原型对象。除了`Object.prototype`，因为Object.prototype是JS世界里的万物之源，世界尽头。`Object.prototype.__proto__`为`null`。原型对象之间的链接靠内置属性`__proto__`来链接。所谓的原型链继承，就是通过指定`__proto__`来实现的。
譬如
```
var foo = {'abc': 123}

foo.__proto__ = Array.prototype

foo.push(456)

foo
// [456, abc: 123]
```

也可以这样操作

```
var foo = {'abc': 123}

foo.__proto__ = new Array

foo.push(456)

foo
// [456, abc: 123]
```
两者的差别就是后者多生成了一个实例，原型链中多了一个空数组作为原型对象。根据我们前面说的js对象属性查找逻辑，最终表现都在Array的prototype上找到了push方法。

### prototype 构造函数的原型对象

那么`prototype`是什么呢？我把`prototype`放在最后介绍，就是因为它是最容易混淆的概念，必须先牢固理解前面说的这些。

`prototype`是每个函数/构造器自带的一个属性，指向函数的原型对象。同时函数的原型对象跟别人的不一样，函数的原型对象里一定自带一个属性叫`constructor`,指向函数自己。

当我也是一个充满好奇心的js初学者时，我也经常在浏览器的控制台里点开复杂对象去查看它的prototype。试图通过这个路径去到达继承世界的尽头`null`。

如果你干过跟我一样的事情，在特别一知半解的阶段时可能永远到不了`null`,因为**constructor的prototype的constructor指向他自己**, 你可以在chrome浏览器里无穷无尽地展开下去。。。。

prototype是Constructor的内置小老弟，不是instance的，不是每个instance都会自带prototype。

它们的关系用图来表达就是这样

![IMAGE](/pimg/E11FF682D98442892BA7F3385CF4EB52.jpg)

写一段程序来追踪原型链的链式指向关系

```
  function traceProto (someJsStuff) {
    while (someJsStuff.__proto__ !== null) {
      console.log(someJsStuff)
      console.log('has a proto: ', someJsStuff.__proto__);
      console.log('has a constructor: ', someJsStuff.__proto__.constructor);
      console.log('which is ' + (
        someJsStuff.constructor === someJsStuff.__proto__.constructor ? '' : 'not '
        ) + 'get from its proto constructor')
      someJsStuff = someJsStuff.__proto__;
    }
  }
```

ES6里增加了Object.getPrototypeOf和Object.setPrototypeOf
所以这里的__proto__指向都是原型对象本象，改了它就改了原型对象（所以一般不让你改）

_那么对于原型链的解读大概就到这里了，下一个主题会是js中的继承。_

预热一个知识点

如何检查是否继承关系？
1. instanceof [可以检查目标构造函数的prototype是否存在于实例的原型链中任何位置](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/instanceof)
2. 父类构造函数的prototype.isPrototypeOf(子类实例)
3. 检查子类实例是否有父类实例所有的属性和方法。

---

彩（踩）蛋问题
---
已知:
  1. 构造函数执行时的this指向一个新对象
  2. Function.prototype.bind可以改变Function函数执行时this的指向

那么, 可不可以通过`bind`强行改变`new`一个构造器时`this`的指向呢？

来，我们让矛和盾打一架，上实验
```
var obj = {'hoho': 'new谁都是我啦'}

function Factory() {
  this.haha = '进工厂改造啦'
  console.log(this === obj)
}

var foo = new (Factory.bind(obj))
// false
obj.haha
// undefined
```
emmmmm....`bind`完败

查了一下[MDN上的解释](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Function/bind),其中包含了针对我上面这种情况的说明，简单来说就是当`new`的时候`bind`谁作为`this`（也就是bind函数第一个参）是忽略不计的。但是bind函数仍然可以提供后面的其它参数的绑定。
<blockquote>
  绑定函数适用于用new操作符 new 去构造一个由目标函数创建的新的实例。当一个绑定函数是用来构建一个值的，原来提供的 this 就会被忽略。然而, 原先提供的那些参数仍然会被前置到构造函数调用的前面。
</blockquote>

所以拓展出来的骚操作果然还是不能在底层老大哥面前头铁。

彩蛋2
--- 
世界尽头是这样的
![IMAGE](/pimg/00B5419DB67580A1C74E2439093F214B.jpg)

Object.prototype的原型对象`__proto__`由getter负责返回`null`
setter负责保护不让你改
如果把Object当作一个实例，它的原型对象指向一个引擎层的函数。

