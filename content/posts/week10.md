---
date: 2019-02-12T10:00:30+08:00
tags: 
- javascript
- fiber
- react
categories:
- 周记
- 翻译
title: React Fiber入门理解
---

_Joe按：本来想看一下fiber的官方资料和源码，发现这篇[Fiber架构导读](https://github.com/acdlite/react-fiber-architecture)作为intro讲得非常好，国内又没有什么满意的翻译，就自己翻了一下。_

### Fiber的目标
摘自官网[CodeBase Overview]([https://reactjs.org/docs/codebase-overview.html\#fiber-reconciler](https://reactjs.org/docs/codebase-overview.html#fiber-reconciler))

我翻译一下fiber的架构大概旨在解决哪些长期的问题

* 将可被打断的工作进行分片的能力
* 在进程中将工作分优先级执行、重定位、和重用的能力
* 在父子进程间来回切换以支持React视图渲染的能力
* 处理`render()`函数返回多个元素，破除单根限制的能力
* 更好地支持错误处理

### 译文:

React Fiber是React核心算法正在进行的重新实现。它是React团队两年多的研究成果。
React Fiber的目标是提高其对动画，布局和手势等领域的适用性。它的主体特征是增量渲染：能够将渲染工作分割成块，并将其分散到多个帧中。

其他主要功能包括在进行更新时暂停，中止或重新使用工作的能力;为不同类型的更新分配优先权的能力;和新的并发能力。

### 关于这个文档
Fiber引入了几个新颖的概念，很难通过查看代码来完成。这个文档是我们在React项目中随着Fiber实现的一系列笔记开始的。随着它的发展，我意识到它也可能成为其他人的有用资源。

我会尝试尽可能使用最普通的语言，并通过明确定义关键术语来避免行话。在可能的情况下，我也会大量连接外部资源。

请注意，我不在React团队，也不会从任何权威机构发言。这不是一个正式的文件。我已经要求React团队的成员对其进行检查以确保准确性。

这也是一个正在进行的工作。Fiber是一个正在进行的项目，在完成之前可能会经历重大的重构。我也试图在这里记录它的设计。非常欢迎改进和建议。

我的目标是在阅读本文档之后，您将会理解Fiber的实施情况，并且甚至最终能够参与React的贡献。

### 前置条件
我强烈建议您在继续之前熟悉以下资源：

[React Components, Elements, and Instances](https://facebook.github.io/react/blog/2015/12/18/react-components-elements-and-instances.html) - “Component”通常是一个重量级的术语。牢牢掌握这些术语至关重要。

[Reconciliation](https://facebook.github.io/react/docs/reconciliation.html) - 对React reconciliation算法的上层描述。

[React基本理论概念](https://github.com/reactjs/react-basic) - 对React概念模型的描述。其中一些内容在第一次阅读时可能没有意义。没关系，随着时间的推移会更有意义。

[React设计原则](https://facebook.github.io/react/contributing/design-principles.html) - 特别注意scheduling部分。它很好的解释了React Fiber的工作原理。

### 复习
如果你还没有看前置条件，请先看完它。

在我们深入研究新的东西之前，让我们回顾一下几个概念。

#### 什么是reconciliation
##### reconciliation 调和
React算法，用来比较两个状态树，以确定哪些部分需要改变。

##### update 更新
用于呈现React应用的数据更改。通常是`setState`的结果。最终导致重新渲染。
React的API的核心思想认为更新会导致整个应用重新渲染。这让开发人员以声明的方式编写，而不用担心如何高效地将应用从特定状态转换到其它状态（从A到B，从B到C，从C到A等等）。

实际上，在每次更改时重新渲染整个应用这种方式只适用于最轻量的应用;在实际的应用中，性能方面的代价非常高昂。
React本身做了优化，可在保持良好性能的同时创建整个应用需要重新呈现的外观。这些优化的大部分内容是reconciliation的一部分。

Reconciliation是被普遍理解为“虚拟DOM”的算法。高级描述如下所示：当你渲染React应用时，会生成一个描述应用的节点树并保存在内存中。然后将该树刷新到渲染环境 - 例如，在浏览器中，将其转换为一组DOM操作。当应用更新（通常通过setState），生成一个新的状态树。新树会与旧树作比对来计算需要更新哪些操作到已渲染的应用中去。

虽然Fiber是对reconciler的底层重构，但是在上层算法部分它们的基本实现是维持不变的，记住下面两个两个关键点：
- 我们假定组件(节点)的构造器不同会带来差异树的巨大变化，React不会再试图去比较他们，会完全将旧节点及其子节点替换掉
- 同级列表中的diff依赖`keys`，要求key保障"可持续性，可预测性，以及同级之间的唯一性"

### Reconciliation vs Rendering

DOM只是React渲染的一种场景，其它主要的场景还有React Native对应的IOS和安卓原生环境（因此"虚拟DOM树"是个不怎么恰当的说辞）
React能支持多种环境下的渲染的原因在于，它从设计上将调和(Reconciliation)和渲染(Rendering)分成两步来处理。reconciler专注计算差异，renderer专注于执行渲染。这也就意味着React DOM和React Native是复用同一套reconciler的。

Fiber重新实现了reconciler，它并不关心渲染层面，虽然renderer针对新架构做了一些适配更改。

### 几个概念声明

- Scheduling 调度
  决定工作什么时候执行的过程
- Work 工作
  需要执行的计算，work通常是一次更新的结果(如setState)

当前React的实现会递归向下遍历整棵树并且在一个事件tick中调更新完的树中所有的`render`方法。将来可以尝试延迟某些更新来避免渲染掉帧。

在React设计中常常会遇见一种场景，当新的数据完成更新后，一些知名库会通过`push`的方式给到发生计算的地方。React始终坚持用`pull`的方式来决定什么地方的计算可以被延迟。

React并不是常规的数据流处理库，而是一个搭建用户界面的库，我们认为它的独特定位在于它需要知道哪些计算在app应用中是立刻紧要需要进行的，哪些是是不需要的。

如果有些元素在视口之外，我们就能滞后它的相关逻辑。如果数据来的比帧率快我们就能把变化综合起来批量处理更新。相对于不那么重要的后台工作(譬如渲染从远端拿到的数据)我们可以提高用户交互产生的工作优先级（譬如点击按钮触发的动画）来避免掉帧。

重点有三

1. 在UI表现上，并不是所有更新都需要立刻响应，事实上无差别更新会造成浪费，掉帧和拖累用户体验。
2. 不同类型的更新行为是有优先级差异的，譬如动画需要比数据更新更快地被响应。
3. 基于'push'的实现需要app(或者开发者)来决定合适执行更新，而基于'pull'的实现则能让框架层(React)来更智能地做这些决策

React目前没有很好地利用scheduling(调度)，每一个tick的会立刻直接更新整棵节点树的`render`。因此重构核心算法以更好的利用scheduling是Fiber版本背后的动因。

接下来看Fiber的实现

---
Fiber是什么？

我们接下来要讨论的是React Fiber架构的核心。Fiber是一个比你的直觉想象得更为底层的抽象。如果你感到理解它很痛苦，不要灰心。继续努力，它最终会看起来合理。(如果你最终get到了它，请来提供这部分的心得)

---
我们已经确立了Fiber的基本目标就是让React更好的利用调度，具体来说我们需要这些能力：

- 暂停、恢复工作
- 给不同的工作分配不同的优先级
- 重用先前已完成的工作
- 丢弃不再需要的工作

实现以上任何一项都需要先找到一种方式将工作分成更小的单元。某种意义上说，那就是fiber（纤维）。我们用fiber表示工作的最小单元。

更深入一点地说，让我们回到React"组件即函数"这个概念上，通常表达为:`v = f(d)`

它的意思是渲染一个React应用就像调用一个函数，这个函数里又包含对其它函数的调用。这个类比对理解fiber很有帮助。

我们的电脑通常会通过调用栈来追踪程序的执行。当一个函数被调用，调用栈上便会增加新的一帧。那个调用栈的帧代表那个函数要执行的工作。

当处理UI的时候，问题在于一次性要执行太多的工作，这样会让动画掉帧看起来卡顿。而且有时候工作会被新进来的更新取代掉，这时候就不需要去执行它的更新。这就是UI组件和函数之间的矛盾之处，因为通常UI组件比函数有更具体的关注点。

较新的浏览器(以及React Native)实现了一些API接口来协助处理这个问题：`requestIdleCallback`会在系统处于空闲阶段的时候计划调用一个底层的函数，`requestAnimationFrame`会在下一个动画渲染帧中会计划调用一个上层的函数。问题在于，如果想使用这些API，你需要将渲染工作分解成一个个增量单元。如果你只依赖于调用栈来执行工作，它就会工作到栈空掉为止。

如果我们可以通过自定义调用栈的行为来优化UI渲染是不是很棒？如果我们可以任意中断调用栈并且手动操作调用栈上的帧是不是很棒？

这就是React Fiber的目的。Fiber就是专门为React组件重写的调用栈。你可以把单个fiber想象成一个虚拟调用栈帧

重写调用栈的好处在于你可以将调用栈里的帧缓存在内存中自由控制它们的执行。这一点对于我们实现调度的目标至关重要。

除了调度之外，手动操作调用栈的帧也解锁了一些潜在的特性，譬如并发和错误处理边界，我们会在之后的部分探讨这些问题。

下一部分我们将继续讨论fiber的架构

### fiber的构造
具体条件下，fiber是一个JS对象，它包含一个组件(component)的信息、输入和输出。

fiber相当于一个调用栈的帧，也相当于一个组件的实例。

这里有一些fiber的重要字段(并不详尽)。

#### type和key

type和key对于一个fiber来说有着和对React元素一样的目的。（事实上当通过一个元素创建一个fiber的时候这两个字段会直接被复制过来）

type描述fiber对应的组件类型。对于react组件来说，type就是构造函数或组件的类本身。对于原生组件来说(譬如div, span等)，type则是一个字符串。

概念上讲，type应该是调用栈的帧所追踪的那个执行函数

key是用于在调和(reconciliation)阶段决定一个fiber是否能被复用的。

#### child和sibling

这两个字段是其它fiber的引用，描述fiber之间的树型结构关系。

子fiber相当于组件`render`方法的返回值。于是在下面例子中

```
function Parent() {
  return <Child />
}
```

Parent组件的子fiber相当于Child组件

sibling字段对应那些`render`方法返回多个子节点的情况(Fiber带来的新特性！)

```
function Parent() {
  return [<Child1 />, <Child2 />]
}
```
子fiber们形成一个单向链表，头部是第一个子fiber, 在这个例子中，`Parent`的child是`Child1`,`Child`的sibling是`Child2`

回到我们的函数类比中，你可以把子fiber看作尾递归调用。

#### return
return fiber指向程序处理完当前这个fiber之后改返回到的那个fiber。概念上来说和调用栈帧返回的地址是一样的。你也可以把它理解为父级fiber。

如果一个fiber有多个子fiber，每个子fiber的return fiber都是父fiber。所以对于我们上一个例子来说，`Child1`和`Child2`的return fiber都是`Parent`

#### pendingProps and memoizedProps
概念上来说，props是一个函数的参数。fiber的pendingProps在执行开始之前设置，而memorizedProps在执行结束之后设置。

当传入的pendingProps与memorizedProps相同的时候，它表示fiber之前的输出可以被重用，以避免不必要的工作

#### pendingWorkPriority
表示fiber工作的优先级。`ReactPriorityLevel`模块列出了不同优先级代表的不同内容。
除以0表示NoWork的特例之外，数字越大优先级越低。例如，您可以使用下面这个函数来检查fiber的优先级是否至少与给定级别一样高：

```
function matchesPriority(fiber, priority) {
  return fiber.pendingWorkPriority !== 0 &&
         fiber.pendingWorkPriority <= priority
}
```

这个函数仅用于说明；它实际上并不是React Fiber代码库的一部分。

scheduler使用优先级字段来搜索下一个要执行的工作单元。这个算法将在以后的章节中讨论。

#### alternate 替换

##### flush
清出fiber用于将其输出到屏幕上

##### work-in-progress
未完成的fiber；概念上来说是一个还未被返回的调用栈帧。
任何时候，对于一个组件实例来说最多有两个fiber对应自己: current, flushed fiber和work-in-progress fiber。

current fiber的alternate是 work-in-progress fiber，work-in-progress fiber的alternate是current-fiber

fiber的alternate是通过`cloneFiber`函数延迟创建的。如果fiber存在alternate，`cloneFiber`会试图复用它以避免总是创建新对象，从而减少分配。

`alternate`理应被视作一个具体实现细节，但它在源码中出现得太频繁以至于有必要在这里讨论一下。

#### output
#####host component
react应用的叶子节点。它们是对应不同渲染环境的(例如在浏览器app中，它们是`div`, `span`)。在JSX中，它们用小写tag标签名表示。
概念上讲，fiber的output就是函数的返回值
每个fiber最终会有output输出，但是output仅由`host components`在叶子节点处创建，然后通过fiber树传递。

output就是最终呈现给渲染器的结果所以它能将变化全部清出到渲染环境中。至于如何将output的内容创建、更新那就是渲染器的事了。

### 未来
目前就是这样，但是这个文档还远远没有完成。以后的部分将介绍在整个生命周期中使用的算法。要涵盖的主题包括：

- scheduler如何找到要执行的下一个工作单元。
- 如何通过fiber树跟踪和传播优先级。
- scheduler如何知道何时暂停和恢复工作。
- 工作如何被清出并标记为已完成。
- 副作用（如生命周期方法）是如何产生的。
- 协程是什么以及如何用它来实现上下文和布局等功能。
