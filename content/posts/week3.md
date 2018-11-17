---
date: 2018-11-17T15:38:30+08:00
tags: 
- tapable
- webpack
- async
- 源码
categories:
- 周记
title: Tapable源码解读(0)
---
_Joe按：难得没有业务缠身，挖了个源码解读坑给自己填，一次性填不完，分篇来弄。核心代码逻辑部分很多文章已经解释得很好了，作为一个严重不喜欢copy-paste的人，会直接quote/link过去。_

### Tapable是什么?

Tapable是一个webpack的标准类库，webpack的整个生命周期及其开放的自定义插件系统都离不开tapable的支持，Webpack的插件就是Tapable对象，我们可以通过tapable为插件定制钩子。
截止到这篇文字的写作时间，tapable一共有两个released版本，但webpack对它的依赖从2013年持续到今天，可谓是非常稳定且精准的抽象。这也就催生了我阅读它源码的想法。本文解读的对象为Tapable1.0。

### Tapable有什么用?
Tapable的核心作用就是管理插件，tapable不关心用户将什么事件挂载到什么处理流程里，这都是具体业务逻辑。tapable关心的是**几种多个事件挂载同一个挂载点上如何运作的模式**，也就是applyPlugins的逻辑。

所以我们先来看一看这两篇文章，了解一下所有的钩子都是怎么使用的和大致实现逻辑。[1.0使用导读](https://www.codercto.com/a/21587.html)

顺便放上[0.2导读和webpack事件流机制](https://segmentfault.com/a/1190000008060440)供对比

API简单概括一下就是

Sync系列

- SyncHook // 普通同步执行，大部分webpack钩子的执行逻辑
- SyncBailHook // 保险式执行，用于加保险栓，一旦return东西就不往下。webpack4中的entryOption， shouldEmit， optimizeChunks就是这类钩子
- SyncWaterfallHook // 瀑布流式执行，顾名思义，管道操作，每一个钩子的入参是上一个的返回值。 代表人物assetPath
- SyncLoopHook // 循环钩子，会执行到没有返回值为止。

Async系列

- AsyncSeriesHook // 普通串行，把异步当同步处理，webpack4中的optimizeTree， optimizeChunkAssets就是这类钩子
- AsyncSeriesBailHook // 保险式串行执行，如果过程中返回任意值则中止执行。
- AsyncSeriesWaterfallHook // 瀑布流式串行执行，顾名思义，管道操作，每一个钩子的入参是上一个的返回值。
- AsyncParallelHook // 普通并行，全部异步执行一遍， webpack4中的make就是这类钩子，为了让makefile非阻塞地并行执行来提高编译速度
- AsyncParallelBailHook // 保险式并行执行，取最早来的一个返回值。

想知道具体哪些钩子是什么Hook类型，参照官方文档的说明[compiler-hooks](https://webpack.js.org/api/compiler-hooks/)和[compilation-hooks](https://webpack.js.org/api/compilation-hooks/)

看完了会发现就webpack的case而言同步模式能解决大部分应用场景，但tapable还是提供了更多的处理模式，1.0的Tapable相比0.2更清晰地将自己定义为钩子管理工具，让人更方便地将其运用到"*多事件在同一挂载点触发的执行顺序/输出结果*"业务逻辑中去。

清楚了目标和它所解决的问题之后，来学习一下它的构建方式深入一下源码吧。

### Tapable源码探索
看源码嘛，可以广度优先，也可以深度优先，和学习一样，无论你选择哪个方向，它们都没法一条道走到黑，这两种思路会相互交错着帮助你更全面地理解源码。我的入手方式可能比较偏深度优先，从一个切面挖下去向横向的公共层去摸索，一点点搭建模块设计上的全局观。

挖掘路径,先从__test__用例入手，
先来看第一个用例

```
it("should use same name or camelCase hook by default", () => {
        const t = new Tapable();
        t.hooks = {
            myHook: new SyncHook()
        };
        let called = 0;
        t.plugin("my-hook", () => called++);
        t.hooks.myHook.call();
        t.plugin("myHook", () => (called += 10));
        t.hooks.myHook.call();
        expect(called).toEqual(12);
    });
```
Tapable.js 测试用例演示了基类Tapable和SyncHook类如何链接
自动转命名
这一个切面其实涉及的横向层很多，简单说一说读到的几个信息

- 1 Hooks是挂在Tapable构造出来的实例上的一个属性，可以挂载不同的hook
- 2 `t.hooks`用于储存所有类型的hook，至于里面具体的hook，0.2时期就是一个简单的数组，通过不同的执行逻辑去遍历，1.0重构过后hook换成了Hook类生成的实例，SyncHook就是其中的一个Hook类。
	- 2.1 SyncHook继承自基类Hook，子类与基类的不同之处在于，SyncHook覆盖了基类的tapAsync和tapPromise方法，如果由于使用不当调到了这两个方法则直接抛出错误。同时Hook基类的compile方法不应该直接被调用，从设计上应该通过子类调用对应的工厂函数进行覆盖。
	```
	// SyncHook.js
			class SyncHook extends Hook {
	    tapAsync() {
	        throw new Error("tapAsync is not supported on a SyncHook");
	    }

	    tapPromise() {
	        throw new Error("tapPromise is not supported on a SyncHook");
	    }

	    compile(options) {
	        factory.setup(this, options);
	        return factory.create(options);
	    }
	}
	```
	- 2.2 子类的代码工厂函数也不是直接引用的，它也是通过工厂函数的基类去继承的。
		- 2.2.1 我们再看一下HookCodeFactory是干什么用的？摘核心代码段

			```
			// HookCodeFactory.js
			class HookCodeFactory {
			    constructor(config) {
			        this.config = config;
			        this.options = undefined;
			        this._args = undefined;
			    }

			create(options) {
			        this.init(options);
			        let fn;
			        switch (this.options.type) {
			            case "sync":
			                fn = new Function(
			                    this.args(),
			                    '"use strict";\n' +
			                        this.header() +
			                        this.content({
			                            …
			                        })
			                );
			                break;
			            case "async":
			                fn = new Function(
			                    this.args({
			                        after: "_callback"
			                    }),
			                    '"use strict";\n' +
			                        this.header() +
			                        this.content({
			                            …                        
						})
			                );
			                break;
			            case "promise":
			                let code = "";
			                    …
			                break;
			        }
			        this.deinit();
			        return fn;
			    }

			```
			这里的底层支持是[Function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function)类，HookCodeFactory用了一种我们不常用的创建函数的方式，像`eval`那样用字符串创建函数，所以这个工厂类的作用是可编程化地用代码写代码。乍一看用代码写代码会是一种调试灾难，但仔细想想单元测试不就是为了解决这类问题而生的吗？可以对照snapShot里面的文件来探究在哪些不同case下生成了哪些代码。这些在`__test__.HookFactory.js`里都有，注意使用expect.[toMatchSnapshot](https://jestjs.io/docs/en/snapshot-testing)的地方就是了。创建代码的配置项里的`type`有这样几个枚举值`sync`,`async`,`promise`。以`type: ‘sync’`为例，打印出用于函数生成的字符串如下
			```
			// header部分
			"use strict";
			var _context;
			var _x = this._x;
			// content部分
			var _fn0 = _x[0];
			var _result0 = _fn0(options);
			if(_result0 !== undefined) {
				return _result0;
				;
			} else {					var _fn1 = _x[1];
				var _result1 = _fn(options);
				if(_result1 !== undefined) {
					return _result1;
			;
				} else {
				}
			}
			```
			一个函数由header和content组成，HookCodeFactory类中不包含content方法，也就是说这是个必须拓展才能使用的设计，content方法由子类提供。而子类对content的编译又需要调用HookCodeFactory的callTap系列方法来生成不同逻辑的代码段。这样分离设计的原因是什么？我们先保留疑问。从目前看到的信息来看，
			```
			// HookCodeFactory.js
			header() {
			        let code = "";
			        if (this.needContext()) {
			            code += "var _context = {};\n";
			        } else {
			            code += "var _context;\n";
			        }
			        code += "var _x = this._x;\n";
			        if (this.options.interceptors.length > 0) {
			            code += "var _taps = this.taps;\n";
			            code += "var _interceptors = this.interceptors\n";
			        }
			        for (let i = 0; i <this.options.interceptors.length; i++) {
			            const interceptor = this.options.interceptor[i];
			            if (interceptor.call) {
			                code += `${this.getInterceptor(i)}.call({this.args({
			                    before: interceptor.context ?"_context" : undefined
			                })});\n`;
			            }
			        }
			        return code;
			    }
			```
			Header里负责定义全局变量`_x`，`_context`, 然后根据`options`传的`tap`和`interceptors`生成`_taps`和`_interceptors`
			Contents里负责处理几种不一样的运行逻辑，CallTapSeries,CallTapParallel, CallTapLooping, 这里也可以对应到0.2版本的几运行机制的实现，一下子逻辑抽象代码自动化了。

		- 2.3 compile函数到底有什么用？compile里调用完成了HookCodeFactory工厂函数的配置和调用，返回了一个编译后的真实Function，这里用了懒加载技术
		```
		// Hooks.js
		function createCompileDelegate(name, type) {
		    return function lazyCompileHook(...args) {
		        this[name] = this._createCall(type);
		        return this[name](...args);
		    };
		}

		Object.defineProperties(Hook.prototype, {
		    _call: {
		        value: createCompileDelegate("call", "sync"),
		        configurable: true,
		        writable: true
		    },
		    _promise: {
		        value: createCompileDelegate("promise", "promise"),
		        configurable: true,
		        writable: true
		    },
		    _callAsync: {
		        value: createCompileDelegate("callAsync", "async"),
		        configurable: true,
		        writable: true
		    }
		});

		```
- 3 `t.plugin`方法来自于`Tapable.prototype.plugin`, 后面的钩子名会被自动转换成camelCase，然后推入`t.hooks`对象底下对应钩子名属性的hook实例管理,（这也是这组用例主要在测的内容）
- 4 断言call值为12，证明了第二次执行`myHook`钩子的时候，之前两个`plugin`的函数都有被执行。

---
时间关系，戛然而止。下周再作整理。


[Compiler和Compilation对象](https://www.jianshu.com/p/e3d97913331a)