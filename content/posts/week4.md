---
date: 2018-11-24T14:29:30+08:00
tags: 
- javascript
- 模块化
- 历史
categories:
- 周记
title: 捋一捋javascript模块化演进史
--- 
_Joe按: 入行晚，上来就是vue+webpack，所以听人扯到AMD, RequireJS, CMD, CommonJS这些词总有种模凌两可的感觉。所以这次接机把这些名词的概念都理清楚。希望读完这篇的你和我一样，缕清这些问题，这些都是个啥?我们为什么曾经需要这些?为什么现在又不需要这些了?_

### 为什么要模块化?

对于任何一种软件开发的形式来说，模块化都是一种必要的能力。对于合理的模块化而言，主要作用有那么几点：
1、复用性：使代码结耦。在不同模块互相组合时，模块内的变量不会污染到模块外，模块间的命名不会冲突。帮助实现代码的高内聚低耦合。
2、可读性：通过文件上的分离，增强了代码的可读性。
3、可维护性：模块化也是单元测试的前提，划分了可测单元。单元测试对代码的意义不再赘述。

### 在JS领域，模块化的从0到1

早期，由于js动态语言的特性，所有js文件都会被加载到同一个环境（也就是浏览器）里执行。

所有全局变量都直接挂在全局对象window下访问。这就产生了命名冲突问题，后声明的覆盖先声明的。

这样的条件下，但凡想写出一点可复用的代码，都面临两个问题

- 如何避免被别人的命名覆盖？
- 如何防止覆盖别人的命名？

第一个问题似乎比较容易，只要在自己的作用域内，覆盖别人的命名就好了。
第二个问题的答案也不难想到，只要把对内部变量的访问，通过接口向外暴露。这样就保护了外面的变量被内部的变量覆盖。

听着不错，可是接口是什么呢？接口通常还是一个Function函数，在js这个万物皆对象的社会里，接口命名和别人冲突了又怎么办呢？

这时候可以一刀两断地切分，只生成作用域不对其命名。由组织代码的调用方去管理命名，这样便让冲突的情况变得可控了。

所以能不能实现这样一块作用域呢？

JS中的匿名闭包就可以实现，俗称IIFE(Immediately Invoked Function Expression)立即执行函数模式。

上代码

```
var Module = (function(){
    var _private = "safe now";
    var foo = function(){
        console.log(_private)
    }

    return {
        foo: foo
    }
})()

Module.foo();
Module._private; // undefined
```

从特性上来讲，这和闭包的定义很相像。

这里重温一下什么是闭包，一句话解释。_*如果一段代码执行结束之后，其词法作域仍可被代码段外部访问到，那么这段作用域就形成了一个闭包。*_

闭包是js的特性，也是js模块化开发的基石。IIFE模式完成了js领域模块化从0到1的演进。

### 从理论到工程化

有了闭包的模式，对于任何一段想被复用的代码来说，似乎它都应该把自己包在一个函数里。这就开始面对很多工程问题了。

譬如我们写代码为了结耦，把代码从文件上分离开了，那么对于调用的那方来说，就需要请求很多小文件。请求数变多了，那么多请求，如果串行执行，怕互相阻塞效率变低。如果并行执行，那万一有的模块文件之间有明确的先后依赖关系又该如何管理呢。

面对这样的问题啊，html提供的能力似乎不够用了。但好在JS是一门正经语言，由JS来接管模块的加载就是了。

举个🌰，LABjs

```
script(src="LAB.js" async)
```

```
$LAB.script("framework.js").wait()
    .script("plugin.framework.js")
    .script("myplugin.framework.js").wait()
    .script("init.js");
```

LABjs的风格有点像是强化了html头上那堆script的能力，使它们能更好的管理顺序和并行加载了。

相比这种偏html的风格，YUI的风格看起来则更接近‘由JS主导的模块化管理’。

```
// hello.js
YUI.add('hello', function(Y){
    Y.sayHello = function(msg){
        Y.DOM.set(el, 'innerHTML', 'Hello!');
    }
},'3.0.0',{
    requires:['dom']
})
```

```
// main.js
YUI().use('hello', function(Y){
    Y.sayHello("hey yui loader");
})
```

<quote>这里的Y就是一个一个强沙箱，所有依赖模块通过 attach 的方式被注入沙盒，即在当前 YUI 实例上执行模块的初始化代码，使得模块在当前实例上可用</quote>

```
// Sandbox Implementation
function Sandbox() {
    // ...
    // initialize the required modules
    for (i = 0; i < modules.length; i += 1) {
        Sandbox.modules[modules[i]](this);
    }
    // ...
}
```

形参Y应该就是指YUI()生成的实例。通过YUI.add添加具名的模块注入逻辑，通过YUI()实例化，通过.use定义如何使用注入的模块。从而耍出这样的操作

```
YUI().use('module1', 'module2', 'module3', function(Y) {
    // you can use all this module now
});
```

YUI同时也支持YUI-Combo的方式将多个资源合并成一个，进行压缩和混淆。在这个方案上我们可以看到现代前端构建工程化的一些雏形。

### 后来，JS走出了浏览器

<quote>2009年，有人讨论将 JavaScript 引入服务器端。因此 ServerJS 诞生了。随后，ServerJS 将其名称改为 CommonJS 。
CommonJS 不是一个 JavaScript 库。它是一个标准化组织。它就像 ECMA 或 W3C 一样。ECMA 定义了 JavaScript 的语言规范。W3C定义了 JavaScript web API ，比如 DOM 或 DOM 事件。 CommonJS 的目标是为 web 服务器、桌面和命令行应用程序定义一套通用的 API 。
0</quote>

Node.js遵循了CommonJS的规范，让JS走出了浏览器端，在CommonJS规范下所有JS代码的加载都是同步的。

每个在nodejs环境下执行的js文件都是一个单独的模块，根据官网的解释，nodejs会这样去包装单文件js模块

```
(function(exports, require, module, __filename, __dirname) {
// Your module code actually lives in here
});
```

也就是说node环境下执行的js文件跟浏览器环境主要差别就是注入的这几个变量。
- module: 表示本模块的对象，表达了本模块的加载情况，模块间关系，分配到的全局id等信息
- exports: module.exports 的引用
- require: 用于引入其它模块的方法
- __filename: 当前文件路径名
- __dirname: 当前文件夹路径名
- global: 相当于浏览器环境下的window，全局变量

CommonJs 非常巧妙地解决了服务器端的模块化加载问题。但是这种操作对浏览器来说却有一个无法回避的瓶颈。对于浏览器环境来说，加载一个个js模块意味着请求不同的资源，而对于服务器来说，只是本地IO微不足道的开销。

假设我们在浏览器端也想享用类似CommonJs这样优雅的模块化书写模式，该怎么办呢？

### 尝到了甜头后回到浏览器，什么是AMD? 什么是CMD?

在网络传输和本地IO效率差距悬殊的今天，首先不可回避的一点是，浏览器的资源管理依旧是异步的，与YUI的解决方案本质上并没有太多不同。作为一种规范，AMD和CMD所关注的，或者说想重新定义的是书写方式的问题。

从诞生年代来说，他们都在CommonJS和NodeJS之后，所以从书写上都受其影响，AMD和CMD在书写上的争议体现了两种思想的不同。下面我们具体说说不同。


假设在浏览器中使用CommonJs的范式去写代码

```
//CommonJS Syntax
var Employee = require("types/Employee");

function Programmer (){
    //do something
}

Programmer.prototype = new Employee();
```

显然这是不work的，因为require以异步的方式在浏览器环境下请求了一个资源。执行new Employee的时候Employee还未接受到加载完的模块。

JS中怎样管理异步行为呢？几乎不用思考可以想到的是，古法回调。。。

所以AMD主张这样干

```
//AMD Wrapper
define(
    ["types/Employee"],  //依赖
    function(Employee){  //这个回调会在所有依赖都被加载后才执行
        function Programmer(){
            //do something
        };

        Programmer.prototype = new Employee();
        return Programmer;  //return Constructor
    }
)
```
AMD定义了一个标准方法define，去确保回调发生在依赖的资源加载完成之后。

在AMD规范中，require是一个用来取模块exports的方法，实际书写起来会是这个样子
```
define(
    ['require', 'dependency1', 'dependency2'],
function (require) {
    var dependency1 = require('dependency1'),
        dependency2 = require('dependency2');

    return function () {};
});
```

写法上AMD主张提前声明

```
define(['a','b'], function(a, b) {
  // do sth
})
```

而CMD主张就近声明, 更接近CommonJS

```
define(function(require, exports, module) {
  var a = require('a')
  var b = require('b')
  // do sth
  ...
})
```

写法范式上来说，两者互相都做了一些兼容。譬如AMD中也可以使用较为接近CommonJS的写法，只是通过语法糖转译来实现。
例如

```
define(function (require) {
    var dependency1 = require('dependency1'),
        dependency2 = require('dependency2');

    return function () {};
});
```
会转成

```
// parse out require...
define(
    ['require', 'dependency1', 'dependency2'],
function (require) {
    var dependency1 = require('dependency1'),
        dependency2 = require('dependency2');

    return function () {};
});
```

无论是AMD还是CMD，共同特点都是要预先加载完所有依赖文件。但分歧在于CMD主张只下载，不执行，AMD主张预先下载并且执行，也许这才是本质核心区别。

AMD基本由[RequireJS](https://github.com/requirejs/requirejs)作为载体，CMD基本由[SeaJS](https://github.com/seajs)作为载体。

这一趴的细节和讨论可以看看著名的文章[前端模块化开发那点历史-玉伯](https://github.com/seajs/seajs/issues/588) 很有意思

### 纷争过后，大势所趋的新方向

写到这里，终于我们迎来了现代前端开发技术栈中最有代表性的一趴。

AMD和CMD的做法始终会用包裹+依赖注入的方式去写代码。而这样的范式在browserify和webpack面前显得没有优势可言。纷争的结束往往不是因为某一方的逐渐获胜，而是因为第三方的降维打击。

Browserify问题的方式非常彻底，既然试图在浏览器端写nodeJs范式的模块，那你写你的，中间加一层编译就是了嘛。browserify通过词法分析依赖关系，直接把所有的依赖代码打包在一起进行压缩混淆。没有什么同步异步的依赖注入管理过程。

同时通过watchify来检测源代码变化做实时重编译。并且通过sourceMap逆向追溯所有的合并压缩和混淆。至此看上去是个很完善的开发方案。用nodejs的方式写代码，watchify监听变化，实时转译成新的文件供预览，调试。

直到2014webpack的诞生，我们终于有了真正大而全的解决方案。

Webpack真正一次性解决了很多问题，包括但不限于遗下

- 兼容各书写范式(CommonJS, AMD, ES6)
- 代码分割，控制文件体积大小，异步分块加载
- Loader加载器和plugin插件机制，让webpack处理任何类型的资源都专向定制，游刃有余
- 完整的开发工具和工作流

譬如loaders的配置
```
// webpack.config.js
module.exports = {
    entry: './main.js',
    output: {
        filename: 'bundle.js'
    }{,
    module: {
        loaders: [{
            test: /\.js$/,
            loader: 'babel-loader'
        }]
    }}
}
```
以上这段配置指定了文件的入口和输出目录、文件名，指定了用babel-loader去解析一切js后缀的文件

同时在webpack的处理下，一切资源皆可require，譬如css和图片, 同时可以配置对应的loader

```
// Ensure the stylesheet is loaded
require('./bootstrap.css');

// get a URL or DataURI
var myImage = document.createElement('img');
myImage.src = require('./myImage.jpg');

// CSS Preprocesser
require('./style.less');
require('./anotherStyle.scss');

// Compile-to-JS Language
var myModule = require('./myModule.coffee');
var myTypedModule = require('./myTypedModule.ts');
```

优雅的插件机制, 通过tapable管理的插件事件流(我另一系列文章会详说)

```
var config = {
    entry: ['webpack/hot/dev-server', './app/main.js'],
    module: {
        loaders: [{
            test: /\.(js|jsx)$/,
            loaders: ['react-hot', 'babel']
        }]
    },
    plugins: [
        //Enables Hot Modules Replacement
        new webpack.HotModuleReplacementPlugin(),
    ],
};
```

[更多webpack的干货](https://github.com/petehunt/webpack-howto)

### 回顾

为了在JS中实现模块化这个美好的目标，我们经历了从闭包(IIFE)->JS文件加载器(LABjs, YUI)->NodeJS自带模块化特性的崛起(CommonJS)->浏览器效仿NodeJS的书写(AMD, CMD)->终极编译解决方案(Browserify, Webpack)

前端的构建依然在进化，只有明确面临的问题，带着目的去追技术，才能正视过往，不惧将来，坚定地前行在新技术的潮流中。

本文示例代码和思维路线来自[js模块化七日谈](http://huangxuan.me/js-module-7day/#/)

--- 

番外，NodeJS官网圈到的重点

1. 在require语法解析一个module的路径名的时候[发生了这些顺序](https://nodejs.org/docs/latest-v6.x/api/modules.html#modules_all_together)

2. [CommonJS这样解决循环引用的执行问题](https://nodejs.org/docs/latest-v6.x/api/modules.html#modules_cycles), 简单概括一下，CommonJS为了确保每个模块不被重复加载，会将已加载的模块缓存到内存里，所以如果出现循环引用，无非是互相拿到了内存中该模块未被加载完的exports的引用。
