---
date: 2018-11-03T22:48:30+08:00
tags: 
- unit test
- 单元测试
- 持续集成
- ci
- karma
- mocha
- travis
- netlify
categories:
- 周记
title: 前端单元测试+持续集成初探
---

***Joe按: 报了个不水的前端课深造一下，之后会每周出一个周报总结一下本周技术。希望能把这个输出整理的习惯坚持下来毕竟(毕竟坚持和努力在我的字典里是那么羞耻的事情)***

---

### 简单看一下第一节课上什么
先上所有涉及的文档  
- [NodeJS Assert API](http://nodejs.cn/api/assert.html) NodeJS断言语法

- [should.js](https://github.com/shouldjs/should.js) 一个断言库

- [mocha](https://mochajs.org/) 一个可在Node环境和浏览器环境上运行的测试框架

- [karma](http://karma-runner.github.io/) 一个测试执行过程管理工具

- [travis CI](https://docs.travis-ci.com/user/languages/javascript-with-nodejs/) 一个持续集成构建项目

前两个知识点都是用来建立断言概念的，所以首先的问题是，什么是断言，有什么作用？根据百度百科[<sub>1</sub>](https://baike.baidu.com/item/%E6%96%AD%E8%A8%80/13021995?fr=aladdin)的定义，表示程序员相信在程序中的某个特定点该表达式值为真。

它的目的是创建更稳定、品质更好且 不易于出错的代码

所以个人理解下来断言是testing case的一种表达形式，测试代码要做的事情就是对各阶段输出的值作出可控的断言，从而提升代码的品质和可回归性。

断言的目的是要抛出一个判断结论，它的实现方式有很多种形式。

NodeJS的assert和should.js库都是用来完成断言这个功能的，前者是语言内置模块，后者是一个js编写的辅助库

#### NodeJS assert模块

assert 模块提供了断言测试的函数，用于测试不变式。
有 strict 与 legacy 两种模式，官方建议只使用 strict模式。
它们主要的区别就是strict模式下任何类型断言判断相等采用的都是`===`而非strict模式用的是`==`

这在js的语境下遵循的含义可以参照[MDN等式比较指南](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Equality_comparisons_and_sameness)

但简单记录概括来说就是

>比较运算 x==y, 其中 x 和 y 是值，产生 true 或者 false。这样的比较按如下方式进行：
> 1. 若 Type(x) 与 Type(y) 相同， 则<blockquote>
> a. 若 Type(x) 为 Undefined， 返回 true。<br/>
> b. 若 Type(x) 为 Null， 返回 true。<br/>
> c. 若 Type(x) 为 Number， 则<blockquote>
     > i. 若 x 为 NaN， 返回 false。<br/>
     > ii. 若 y 为 NaN， 返回 false。<br/>
     > iii. 若 x 与 y 为相等数值， 返回 true。<br/>
     > iv. 若 x 为 +0 且 y 为 −0， 返回 true。<br/>
     > v. 若 x 为 −0 且 y 为 +0， 返回 true。<br/>
     > vi. 返回 false。</blockquote>
     > d. 若 Type(x) 为 String, 则当 x 和 y 为完全相同的字符序列（长度相等且相同字符在相同位置）时返回 true。 否则， 返回 false。<br/>
     > e. 若 Type(x) 为 Boolean, 当 x 和 y 为同为 true 或者同为 false 时返回 true。 否则， 返回 false。<br/>
     > f. 当 x 和 y 为引用同一对象时返回 true。否则，返回 false。</blockquote>
> 2. 若 x 为 null 且 y 为 undefined， 返回 true。
> 3. 若 x 为 undefined 且 y 为 null， 返回 true。
> 4. 若 Type(x) 为 Number 且 Type(y) 为 String，返回比较 x == ToNumber(y) 的结果。
> 5. 若 Type(x) 为 String 且 Type(y) 为 Number，返回比较 ToNumber(x) == y 的结果。
> 6. 若 Type(x) 为 Boolean， 返回比较 ToNumber(x) == y 的结果。
> 7. 若 Type(y) 为 Boolean， 返回比较 x == ToNumber(y) 的结果。
> 8. 若 Type(x) 为 String 或 Number，且 Type(y) 为 Object，返回比较 x == ToPrimitive(y) 的结果。
>    9. 若 Type(x) 为 Object 且 Type(y) 为 String 或 Number， 返回比较 ToPrimitive(x) == y 的结果。
>    10. 返回 false。

具体assert模块的方法如下:

> - .equal/.notEqual 断言是/否相等(用`==`)
> -	.deepEqual/.notDeepEqual 断言是/否深度相等(用`==`)
> -	.strictEqual/.notStrictEqual 断言是/否严格相等(用`===`)
> -	.strictDeepEqual/.notStrictDeepEqual 断言是/否严格深度相等(用`===`)
> -	.throws/.doesNotThrow 断言是/否抛出异常
> -	.rejects/.doesNotReject （promise专用）断言promise是/否reject
> - .ifError 断言返回null或undefined，同时可以捕捉传入Error类型的调用堆栈踪迹。
> - .fail 主动抛出Error，如果没有传入的Error类则抛出AssertionError

由于是js系列最接近底层[native]的语法所以每个方法都看了一遍，所有的框架、库、它们所做的事情就是基于这些方法去拓展更实用的逻辑。

譬如should.js就是一种拓展。

#### shouldJS

简单看了下文档
这是一个js编写的断言库，根据官网解释，它拓展了Object.prototype的方法，因为js里一切皆对象，所以任何对象都可以使用.should方法
同时支持模块化注入和global注入should方法
大多数情况下`.should`和`should`拥有相同的表现

具体功能很强大，在封装了nodeJS底层断言方法的基础上，增加了一些实用的方法，譬如断言对象的属性、key，断言异步行为，断言结果是否为枚举值等拓展了的功能

非常轻量好用易引入。其它的细节可能只能在具体使用中再细究了。

以上两者一个是底层断言模块，一个是拓展库，再工程化一点的形态应该就是#测试框架#了

Mocha就是这样的一个框架

#### Mocha

相比shouldJS而言，它是完整的一套解决方案，有用于描述测试类的方法(describe)，用于描述某一特征用例的方法(it)，用于测试异步行为的方案(done回调)，有从开发环境可以执行的客户端程序(mocha)用于执行所有的匹配test命名的文件并输出测试结果。从工程上自动分离了所有测试代码和业务代码。

同类型对标的选择的还有Jasmine，Jest等，这些都可以根据个人喜好和熟悉程度去选择，譬如引入是不是足够轻量，语法是不是冗长，是否支持ES6，但它们解决的问题是一样的。都是一种框架级别的前端单元测试解决方案。

理解了框架之后，出于对开发体验和更多功能的追求，我们可能会需要Karma这样的工具

#### Karma

Karma可以持续地watch业务代码和测试代码的变化去执行单元测试的逻辑，输出断言结果。你可以配置多个执行环境，譬如某个模块想同时在浏览器环境和NodeJS环境下测试，Karma就能够满足这样的管理需求。

本地化的karma启动方式什么的我就不赘述了看文档链接就是。更多细节其实也需要看配置说明。

Karma可以整合到webpack项目，gulp项目，grunt项目甚至非AMD方式的项目中。官方主要支持的框架有Jasmine, Mocha, QUnit（但其实他们都通过npm package以plugin的方式组合，所以也可以自己写)

Karma支持[Chrome](https://www.npmjs.com/package/karma-chrome-launcher), [PhantomJS](https://www.npmjs.com/package/karma-phantomjs-launcher), [IE Edge](https://www.npmjs.com/package/karma-virtualbox-edge-launcher), [IE 11](https://www.npmjs.com/package/karma-virtualbox-ie11-launcher), [FireFox](https://www.npmjs.com/package/karma-firefox-launcher), [Safari](https://www.npmjs.com/package/karma-safari-launcher), [IOS simulator](https://www.npmjs.com/package/karma-ios-simulator-launcher)等很多环境的模拟，没有一一测过，有真实项目再踩坑了。

#### Travis

有了开发过程中的测试方案之后，对于项目开发的工程化来说，不仅应该包含业务代码和模块的测试代码，它应该有一个可持续集成的方案，来管理代码变动后的发布行为。这个行为可能是单台宿主机对多个环境的部署，部署过程中可以插入很多事情，譬如通知其它相关的应用(webhook), 譬如进行代码的测试(test), 譬如对前端工程文件进行脚本化的构建，压缩，混淆，加密等等。这一整套事情都可以由Travis来完成，travis配置很简单，只要在项目根目录下放.travis.yml文件并根据官网配置去管理对应的流程就行了。

放上[demo](https://github.com/josephmax/exercise3)，Travis会在收到代码更新后自动去执行`package.json`文件里的`test`script, 这样就完成了部署前的自动化测试。

### 记一次跌宕起伏的持续集成方案实(cai)践(keng)

---

说到持续集成啊，本博客就花了我不少时间去跑通持续集成的过程，
简单唠唠踩坑的心路历程。

首先感谢stkevin大牛开发的基于Hugo的UI theme，让我在众多自定义模板中一眼相中了外观。于是clone了下来，项目的搭建过程参照[这篇文章](https://josephme.netlify.com/posts/hugo-guidance/)，里面记录了对项目打包，站内索引，延迟加载和持续集成等问题的解决过程，非常全面。


出版做完实用了github-pages发布，踩到的第一个坑是部署后的资源路径问题，


翻了一通文档后发现**github会把命名为`<username>.github.io`的项目自动配给域名`https://<username>.github.io`, 并且只支持发布`master`分支**


那就意味着打包结果必须输出到根目录下的index.html中，如果没有会回落到README.md去。


这跟我的项目构建方式略有冲突，好吧，删库重建


第二发建了一个名叫blog的项目，这时可以选择在github-pages上发布`master分支下的的/doc目录`了，访问地址为`https://<username>.github.io/blog/`，美滋滋


按理说这样就算部署完了，但是这样有一个缺点，就是每次都要先build再提交。个人是很反感这样的构建方式的，因为git本身是二进制文件存储着开发过程中的所有内容，多次build带来的变化，不该由git去承担这个管理负荷，用人话来说就是**多次build并提交会让.git目录膨胀得比想象中快很多**，可能对纯文本代码来说算不了个事儿，但每次图片类资源都重新处理是不是感觉就上了一个数量级，这样的方案对我来说是**不能忍**的。


那么，相应的解决方案其实就是把构建过程交给部署机去做。这个方案有trade-off, 牺牲的是build环境的一致性（原来都是本机嘛），现在换到远端服务器去了，**万一两边build出来的表现不一样，就要承担处理这个不一致性带来的后果**。


原theme作者也考虑了这个问题，所以提了一嘴netlify。于是我就试了一下netlify，于是新的问题来了。


Netlify会配给你二级域名.netlify.com, 三级域名可自定义，于是我选了https://josephme.netlify.com。
远端机器构建完显示一切正常，但是我当访问构建出来的地址时直接404啥也没有。 Σ(°Д°;


一翻令人头秃的排查过后，发现js文件和img文件按照路径去访问是有的，但是所有html没有。


所以锁定到gulp构建出了问题，顺着gulp深挖下去发现，哦原来html文件的模板资源来自于一个theme的私有子模块，本地是先安装再重新git init提交的，远端自然不会知道子模块这个事。


于是重来，fork了一下子模块，确认我的账号对两个repository都有访问权限，通过`git submodule add <repository地址>`本地安装了子模块再`git push`到库，重新build。( ºωº )


这时居然build failed了。查log报错说Host Key Verification Failed. Σ(°Д°;


又一翻令人头秃的排查无果后，写了邮件给Netlify的人描述了问题。。。大兄弟半夜回复了我，给了我一个对应发布机的deploy key, 说把这个配到github账号的ssh key里就等于授权这台机器去访问你的代码了，如果你的submodule是私有库只能这么干，github限制。


Em…..很棒棒，配完还真的不报这个错了( ºωº )，然鹅一通build过后，还是没有html文件。Σ(°Д°;


但是这对多次从头秃边缘回来的我来说已经不是什么心惊肉跳的事情了，仔细翻了一下log里面有几行报了编译html文件失败，`function partialCache not defined`. Σ(°Д°;


这个报错输入谷歌基本啥也搜不到，通常这种情况我会怀疑错误太过低级，然而一番令人头秃的排查过后依然没有进展 Σ(°Д°;


这时我忽然发现报错来自于hugo框架那一层的某一模块，于是搜了一下这个模块+行数，居然搜到了hugo社区里的一个问题，报了另一个函数 not defined。对应的解决方案是在netlify编译的时候指定hugo版本，因此可见netlify使用的默认hugo版本低于作者开发用的版本，没有这个语法所以报错了。解决方案也很简单，在项目根目录下加上.netlify.yaml文件，并配置hugo版本为跟我本地一样的版本。


最后build，一把梭，完美编译。


记一次跌宕起伏的踩坑，抱一抱头最终没有秃的自己。