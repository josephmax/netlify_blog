---
date: 2019-03-09T18:07:30+08:00
tags: 
- javascript
categories:
- 周记
title: 可能是我写得最好懂的Generator, async await, promise的实践
---

_Joe按：async await这个概念从2016年下半年就有耳闻，起初对它的认知是promise的语法糖，后来ES6普及了，接触到了Generator，又听说async await是基于generator实现的。由于它一直是个语法糖而且标准不太稳定，我没有太关注它。直到今天大家好像已经把它当作现有世界的一部分了，惊觉自己out了，赶紧回头看看清楚它皮囊之下到底是些啥了。_

### 先上官方文档
[Google的官方解释](https://developers.google.com/web/fundamentals/primers/async-functions)

[tc39的官方说明](https://tc39.github.io/ecmascript-asyncawait/)

### 什么是async，它的好处是什么？

我举个没有阅读成本的栗子，每一段都可以贴进chrome console玩一下。
假设我们要串行地执行几个异步事件。
我们简单地封装一个叫`wait`的函数，表示异步工作，done表示它的回调
```
function wait(time, done) {
    setTimeout(done,time)
}
```
于是用上古包菜式的写法是这样
```
function initTask() {
    wait(1000, function task2() {
        console.log('task1 done')
        wait(2000, function task3() {
            console.log('task2 done')
            wait(3000, function () {
                console.log('all tasks done')
            })
        })
    })
}
```
emmm, 代码看起来像被冲击波轰过一样，教科书式的回调地狱。

这种范式的不雅在Promise诞生之后改善了许多

Promise改变了我们的回调地狱般的思考习惯，让我们先把异步事件`wait`用promise的形式封装一下，改成Promise范式
```
function waitPromise(time) {
    return new Promise(resolve => setTimeout(resolve,time))
}
```
那么我们再来改写一下上面的函数，用Promise的方式实现同样功能
```
function initTask () {
    // start task1 stuff
    waitPromise(1000).then(() => {
        console.log('task1 done')
        // start task2 stuff
        return waitPromise(2000)
    }).then(() => {
        console.log('task2 done')
        // start task3 stuff
        return waitPromise(3000)
    }).then(() => {
        console.log('all tasks done')
    })
}
```
可以看到用了Promise之后，promise内部不需要管回调是什么，直接由它的调用函数去关注后续的操作。某种意义上更好地分离了关注度。同时它把原来看起来无穷无尽的回调地狱“拍平”成了固定的一级嵌套，不管后面执行多少promise, 都是一层。这的确可以说是解决了回调地狱。

不过虽说不会无限扩张了，但还是套了一层，代码看起来多少不太整洁。

而这个问题通过async得到了进一步的优化

async不像Promise，它不是一种新的构造器造出来的对象，它更像是语法中的保留字。如果同样的实现用async写，代码看上去就像下面这样。

```
async function initTask() {
    // start task1 stuff
    await waitPromise(1000)
    console.log('task1 done')
    // start task2 stuff
    await waitPromise(2000)
    console.log('task2 done')
    // start task3 stuff
    await waitPromise(3000)
    console.log('all tasks done')
    return
}
```
看起来就跟同步的代码没什么差别了是不是？

async和await的实现一开始只是个议案，也就是只定义了what to do, not how to do。所以如何实现这样的效果呢？
其实从Promise到async，最大的魔法在于generator带来的协程效果。

不了解[Generator](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/function*)的可以看链接文档。

他的重要特性在于"协程"，他可以交出当前函数的执行权去执行另一段代码，而执行完成后还能从刚才暂停的位置继续执行。以下摘自MDN
> 生成器函数在执行时能暂停，后面又能从暂停处继续执行。
调用一个生成器函数并不会马上执行它里面的语句，而是返回一个这个生成器的 迭代器 （iterator ）对象。当这个迭代器的 next() 方法被首次（后续）调用时，其内的语句会执行到第一个（后续）出现yield的位置为止，yield 后紧跟迭代器要返回的值。或者如果用的是 yield*（多了个星号），则表示将执行权移交给另一个生成器函数（当前生成器暂停执行）。
next()方法返回一个对象，这个对象包含两个属性：value 和 done，value 属性表示本次 yield 表达式的返回值，done 属性为布尔类型，表示生成器后续是否还有 yield 语句，即生成器函数是否已经执行完毕并返回。

下面这句话很重要，可以理解为generator函数跟其它协程通信的入口。
> 调用 next()方法时，如果传入了参数，那么这个参数会作为上一条执行的  yield 语句的返回值

我们先来把上面的工作用generator来重写一遍吧
```
function* initTask() {
    // start task1 stuff
    yield waitPromise(1000)
    console.log('task1 done')
    // start task2 stuff
    yield waitPromise(2000)
    console.log('task2 done')
    // start task3 stuff
    yield waitPromise(3000)
    console.log('all tasks done')
}
```
看起来挺像对不对，问题在于generator返回的是一个迭代器，它需要你手动用`next()`去调。于是我们来写个协程函数帮助它自执行吧。

```
function runner(generatorFn) {
    const iterator = generatorFn()
    function runGenerator(arg) {
        const result = iterator.next(arg)
        if (result.done) {
            return result.value
        } else {
          // 如果不是最后一个yield，则把上一个yield的返回结果封装进Promise里再传给下一个yield
            return Promise.resolve(result.value).then(runGenerator)
        }
    }
    return runGenerator()
}
```
这样当你跑`runner(initTask)`的时候神奇的事情就发生了。

现在把runner想象成async关键字，yield想象成await关键字。是不是看起来就没有那么神秘了？

当然这个runner写得比较糙，edge case处理得比较完善的实践可以看[这里](https://gist.github.com/jakearchibald/31b89cba627924972ad6)

那么async和await到底是promise还是generator呢？当然是两者结合拉，generator负责让代码看起来像同步的，promise负责让代码做异步的事情。

### 实战区
最后让我们来一道应试题体验一下event loop的考法吧～

求以下程序输出: 
```
async function async1() {
    console.log('async1 start')
    await async2()
    console.log('async1 end')
}

async function async2() {
    console.log('async2')
}

console.log('script start')

setTimeout(function(){
    console.log('setTimeout')
},0)

async1()

new Promise(function(res) {
    console.log('promise1')
    res()
}).then(function(){
    console.log('promise2')
})

console.log('script end')

```

这种问题万变不离其宗，我们把javascript一次同步事件执行栈称为一个tick(毕竟执行不完就会卡在那里进不了下一个tick, 异步则不是这样)。碰到这样的问题，考点主要是在于你先要理解tick，setTimeout的会把函数的执行顺序丢到特定秒后去，即使是0也会跑到下一个tick去。另外要理解宏任务和微任务的区别，Promise的异步是基于微任务的执行。微任务的队列在每一个宏任务的tick的开头执行。大概情况看这张图吧。

那么不多说了用大脑模仿执行引擎去跑一下题目代码吧。

1. 11行之前都是函数的定义没有执行所以队列里是空的，直到11行出现了第一个console.log。所以我们先输出<font color=green>**script start**</font>。
2. 接下来13行出现setTimeout，这是异步执行栈的标识。也就是说不管里面是什么，至少这是<font color=red>下一个tick的宏任务</font>。
3. 然后17行我们执行了`async1`函数，2行的定义，这里会先输出<font color=green>**async1 start**</font>。紧接着是`async2`里的<font color=green>**async2**</font>。然后因为async2它<font color=red>也是一个异步函数</font>，它的resolve的值async2的返回值，也就是undefined。但是由于await的缘故我们等要到下一个微任务队列里才会拿回它的执行权，我们只能先接着往下看。这里注意了，3行开始就是<font color=red>下一个tick第一个要执行的微任务</font>。
4. 这时出现了19行一个新的Promise，`new`关键字触发了后面函数的立刻执行，所以我们先输出<font color=green>**promise1**</font>，然后发现这个promise立刻就resolve掉自己了,那么它的`then`就是<font color=red>下一个tick第二个要执行的微任务</font>。
5. 最后我们输出26行的<font color=green>**script end**</font>，至此第一个tick运行完毕了。
6. 接下来进入第二个tick，先做微任务队列，再执行宏任务队列.
7. 微任务队列里的第一个任务就是从3行开始执行，await拿到了一个resolve的Promise，于是将输出<font color=green>**async 1 end**</font>。继续执行下一个微任务。
8. 下一个微任务就是我们第4步时候推入的输出<font color=green>**promise2**</font>。
10. 最后执行第二个tick中的宏任务输<font color=green>**setTimeout**</font>。

所以最终输出为:
```
script start
async1 start
async2
promise1
script end
async1 end
promise2
setTimeout
```
不过很遗憾这个答案在chrome环境和node环境下是不同，我把矛盾更具体化成了以下这个例子

```
(async function async1() {
    await Promise.resolve()
    console.log('next tick 1')
})()

Promise.resolve().then(function(){
    console.log('next tick 2')
})
```
其实你执行一下会发现在chrome74里是先1再2的，而在node11.11.0下是先2再1的。

### todo区

为什么呢？留个todo，捋清楚了再更新这篇文章
以下资料留作参考。
[v8团队的解释](<https://v8.dev/blog/fast-async>)

[ES next中async/await proposal实现原理是什么？ - justjavac的回答](https://www.zhihu.com/question/39571954/answer/148420891)

### 踩蛋区
- 找资料的时候发现阮一峰老师在[描述]((http://www.ruanyifeng.com/blog/2015/04/generator.html))"协程与多线程"的时候还被评论区勘误了一波。在此马克一下，毕竟起初这个问题也的确困扰了我。generator的最大特点在于释放执行权和恢复执行，它本身不能够实现并发或异步。

- btw经亲测发现node是从7.6开始支持 async的。