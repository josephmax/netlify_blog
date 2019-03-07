---
date: 2019-03-02T16:00:30+08:00
tags: 
- javascript
- 源码
- react
- redux
- reselect
categories:
- 周记
title: 基于React渲染原理的优化思路总结
---

_Joe按: 这篇是一个handbook式的手抄，有自己的总结，有其它源码导读的片段，还有一些值得一层层楼爬的讨论链接，记得有人说过Vue is easy, React is simple.这个simple背后的是关于范式和最佳实践的不断探讨和整个生态圈的生命力。_

### 为什么虚拟DOM快？
因为虚拟DOM是plain Object, 操作开销是纯js进行的，远小于对DOM的操作开销。React由数据变化驱动，对状态变更前和变更后的两颗虚拟DOM树进行diff，也就是Reconciliation，计算出最小的变动，再 patch 到真实 DOM 上最终完成用户看得到的更新。

### 那么仅这样就够了吗？
显然不是， React 不是万能的，当我们更新一个组件时，整个 reconciliation 至少会经过以下四个阶段

```
|1
|-组件的 props/state 更改(开发者控制)
|2
|-shouldComponentUpdate（开发者控制）
|3
|-计算 VDOM 的更新（React 的 diff 算法会计算出最小化的更新）
|4
|-更新真实 DOM (React 控制)
```

这四个阶段是串行的，每个阶段都可以中断后面的阶段。1、2阶段算是暴露给开发者的API。为了让你帮助React进行更好的性能优化。

### shouldComponentUpdate 和 PureComponent
理论上来说应该为每个使用 class 声明的组件添加 shouldComponentUpdate，否则一旦接受新的 props/state 就可能进行不必要的 re-render。

如果你很清楚每次的 props 或 state 的都是一个指向新引用的对象，那么可以直接使用 PureComponent，PureComponent 已经实现了一套浅比较的（shallowCompare）的 shouldComponentUpdate 的规则。

在React里，shouldComponentUpdate 源码为：
```
// ...
if (this._compositeType === CompositeTypes.PureClass) {
  shouldUpdate = !shallowEqual(prevProps, nextProps) || ! shallowEqual(inst.state, nextState);
}
// ...
```
shallowEqual 的源码为
```
'use strict';

const hasOwnProperty = Object.prototype.hasOwnProperty;

/**
 * inlined Object.is polyfill to avoid requiring consumers ship their own
 * https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/is
 */
function is(x: mixed, y: mixed): boolean {
  // SameValue algorithm
  if (x === y) { // Steps 1-5, 7-10
    // Steps 6.b-6.e: +0 != -0
    // Added the nonzero y check to make Flow happy, but it is redundant
    return x !== 0 || y !== 0 || 1 / x === 1 / y;
  } else {
    // Step 6.a: NaN == NaN
    return x !== x && y !== y;
  }
}

/**
 * Performs equality by iterating through keys on an object and returning false
 * when any key has values which are not strictly equal between the arguments.
 * Returns true when the values of all keys are strictly equal.
 */
function shallowEqual(objA: mixed, objB: mixed): boolean {
  if (is(objA, objB)) {
    return true;
  }


  if (typeof objA !== 'object' || objA === null ||
      typeof objB !== 'object' || objB === null) {
    return false;
  }

  const keysA = Object.keys(objA);
  const keysB = Object.keys(objB);

  if (keysA.length !== keysB.length) {
    return false;
  }

  // Test for A's keys different from B.
  for (let i = 0; i < keysA.length; i++) {
    if (
      !hasOwnProperty.call(objB, keysA[i]) ||
      !is(objA[keysA[i]], objB[keysA[i]])
    ) {
      return false;
    }
  }

  return true;
}
```

shallowEqual 会对两个对象的每个属性进行比较，如果是非引用类型，那么可以直接比较判断是否相等。如果是引用类型，则通过比较引用的地址进行严格比较（===）。
当 props 和 state 是不可突变数据的时候，可以直接使用 PureComponent，PureComponent 非常适合配合 immutablejs 来做优化。

### stateless function
根据官网对它的定义
> This simplified component API is intended for components that are pure functions of their props.

无状态函数组件的表达式是一个纯函数，完全符合 `UI = f(state)` 的公式，纯函数意味着可预测（给定输入可以获得可预测的输出），这样我们在写代码及调试的时候能对组有更好的把握，如果需要引入副作用则请使用 class。

stateless function一般被用来当作展示组件，这样做的好处有：

- 没有 state，没有 ref，没有生命周期，React 还可以避免不必要的内存申请及检查，这意味着更高效的渲染，React 会直接调用 createElement 返回 VDOM
- 代码更简单易读
- 由于不支持 state，会迫使开发者将逻辑组件与展示组件进一步分离
可以作为 render prop 的 prop，或者完成 callback render
- 更方便进行测试

但是要注意，这并不意味着滥用stateless function：

> state should be managed by higher-level “container” components, or via Flux/Redux/etc.

不要让stateless function直接暴露在数据逻辑中，sc的父组件要完成以下行为的控制：
- 渲染时机
- 上层数据

第一点一定要控制好，因为 sc 没有生命周期，只要传入新的 props/state 就会 re-render，而且这意味着，一旦 sc 的父组件更新，sc 就会 re-render。sc作为一个纯函数，放弃了中断后续操作的能力，re-render 就会带来 VDOM的 diff，这会带来一笔开销（这比开销也可能会是更好的选择，也可能不是）。

### 那么stateless function 和 PureComponent到底该怎样取舍呢？

sc 的缺点在于当 props 更新或父组件重新渲染就会 re-render，它没有放弃更新的能力。但与此同时PureComponent 的缺点在于如果 shouldComponentUpdate 为 true，那么相当于多做了两次检查（SCU 一次，re-render 时 diff 一次）
shallowCompare 可能比 diffing 算法耗费更多的时间。

所以一个组件如果经常变更的话，那么 PureComponent多带来的两次检查会让他通常更慢。

所以我们可以根据经验粗略得出以下指导性结论：

- pure component适用于复杂的表单和表格，它们通常会减慢简单元素（按钮、图标）的效率。

- 一般来说，sc 适用于小的组件（就索性让它做 diff，diff 的开销小，反正不变就不会改变真实 DOM）。PureComponent 适用于稍大一点的组件，diff的代价是随着组件的增大而提升，shallowCompare 与组件无关的复杂度无关而是与 props 的数量挂钩，不同的组件和 props 的要根据实际情况来判断。

### 避免props不必要的更改
不要在 render 中重新定义函数
> 无论是编写哪个阶段的`render` 函数，请牢记一点：保证它的“纯粹”（pure）。怎样才算纯粹？最基本的一点是不要尝试在render里改变组件的状态。

很多人喜欢在 render 函数中子组件通过构造一个箭头函数来传递给子组件，但是这样有一个问题就是，每次都会声明一个新的箭头函数，因而每次声明的函数都肯定是不同的，所以就会导致如果你用了 PureComponent 也无法阻止 re-render，如下：

```
class Parent extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      count: 0
    };
  }

  onClick() {
    this.setState({
      count: this.state.count + 1
    });
  }

  render() {
    return (
      <div>
        {this.state.count}
        <Child onClick={() => {
            this.onClick();
          }}
        />
      </div>
    );
  }
}

class Child extends React.PureComponent {
  render() {
    console.log("Child re-render");
    return <button onClick={this.props.onClick}>add</button>;
  }
}
```
解决方案：在 constructor 中 bind，甚至或者是闭包引用一个 self = this ，或者在类声明中直接定义实例属性，都可以做到绑定 this。

### 使用稳定的 key
作为 key 的键应该符合以下条件

> 唯一的：元素的 key 在它的兄弟元素中应该是唯一的。没有必要拥有全局唯一的 key。
  稳定的：元素的 key 不应随着时间，页面刷新或是元素重新排序而变。
  可预测的：你可以在需要时拿到同样的 key，意思是 key 不应是随机生成的。

不要用index作为同级之间的 key，从源码看来那是一个容错行为。但如果列表中的元素被删除或移动，则整个 key 就失去了与原项的对应关系，加大了 diff 的开销。

### store 的设计

####范式化

多数情况下我们的应用是要配合 Redux 或者 MobX 使用的。拿 Redux 举例，Redux store 的组织是一门大学问，Redux 官方推荐将 store 组织得扁平化和范式化，所谓扁平化，就是整个 store 的嵌套关系不要太深，实体之下不再挂载实体，扁平化带来的好处是

> 当某些数据需要在不同页面中复用，就会存在必然重复。例如，可能存在很多 state 部分都要存储同一份“用户评论列表”，这样需要花费很多心思去保障多处“用户评论列表”数据状态一致，否则就会造成页面数据不同步的 Bug;

> 嵌套深层的数据结构，会直接造成你 reducers 编写复杂。比如，你想更新一个很深层次的数据片段，很容易代码就变得丑陋。具体可以参考这篇文章：[如何优雅安全地在深层数据结构中取值](https://zhuanlan.zhihu.com/p/27748589);

> 造成负面的性能影响。即便你使用了类似 immutable.js 这样的不可变数据类库，最大限度的想保障深层数据带来的性能压力，那你是否知道 immutable.js 采用的 “Persistent data structure” 思路，更新节点会造成同一条链儿上的祖先节点的更新。更恐怖的是，也许这些都会关联到众多 React 组件的 re-render;

范式化是指尽量去除数据的冗余，因为这样会给维护数据的一致性带来困难，就像官方推荐 state 记录尽可能少的数据，不应该存放计算得到的数据和 props 的副本，而是将他们直接在 render 中使用，这也是避免了维护数据一致性的困难，并且避免了相同数据满天飞不知道源头数据是哪个的尴尬。

### state vs store
首先要明确，不要将所有的状态全部放在 store 中，其实再延伸一下可以延伸出 render(){} 中的变量，也就是 store vs state vs render，store 中应该存放异步获取的数据或者多个组件需要访问的数据等等，redux 官方文档中也有写什么数据应该放入 store 中。

> - 应用中的其他部分需要用到这部分数据吗？
> - 是否需要根据这部分原始数据创建衍生数据？
> - 这部分相同的数据是否用于驱动多个组件？
> - 你是否需要能够将数据恢复到某个特定的时间点（比如：在时间旅行调试的时候）？
> - 是否需要缓存数据？（比如：直接使用已经存在的数据，而不是重新请求）

而 store 中不应该保存 UI 的状态（除非符合上面的某一点，比如回退时页面的滚动位置）。UI 的状态应该被限定在 UI 的 state 中，随着组件的卸载而销毁。而 state 也应该用最少的数据表示尽可能多的信息。在 render 函数中，根据 state 去衍生其他的信息而不是将这样冗余的信息都存在 state 中。store 和 state 都应该尽可能的做到熵最小，具体的讨论可以看[redux store取代react state合理吗？](https://www.zhihu.com/question/271693121)。而 render 中的变量应该尽可以去承担一个衍生数据的责任，这个过程是无副作用的，可以减少在 state 中产生冗余数据的情况。

### react-redux

这里在说一下 react-redux 的 HOC 触发更新的条件：

这里有两个问题：1. 在 react-redux 中，connect 出来的 HOC 是怎么感知 store 的变化的？2. 什么的变化会触发 HOC 的更新？

先来解决第一个问题：

react-redux 通过 Provider 提供的 context 上的 store，在内部向 store subscribe 了 onStateChange 事件

```
// ...
trySubscribe() {
    if (!this.unsubscribe) {
      this.unsubscribe = this.parentSub
        ? this.parentSub.addNestedSub(this.onStateChange)
        : this.store.subscribe(this.onStateChange)
 
      this.listeners = createListenerCollection()
    }
  }
// ...
```
只要派发了 action，就会触发一次 onStateChange 事件，HOC 就能感知 store 的更新再根据 onStateChange 的结果决定是否要 update。再来看 onStateChange 的源码：

```
// ...
onStateChange() {
    this.selector.run(this.props)

    if (!this.selector.shouldComponentUpdate) {
        this.notifyNestedSubs()
    } else {
        this.componentDidUpdate = this.notifyNestedSubsOnComponentDidUpdate
        this.setState(dummyState)
    }
}
// ...
```

是由 run 这个是个方法来每次决定每一次的 shouldComponentUpdate

```
// ...
function makeSelectorStateful(sourceSelector, store) {
  // wrap the selector in an object that tracks its results between runs.
  const selector = {
    run: function runComponentSelector(props) {
      try {
        const nextProps = sourceSelector(store.getState(), props)
        if (nextProps !== selector.props || selector.error) {
          selector.shouldComponentUpdate = true
          selector.props = nextProps
          selector.error = null
        }
      } catch (error) {
        selector.shouldComponentUpdate = true
        selector.error = error
      }
    }
  }

  return selector
}
// ...
```

也就是说，当 store 更新的通知到来，就会调用 sourceSelector 重新计算一次结果，与上次缓存的结果进行 === 比较。如果发现比较不相等，即需要更新，shouldComponent 置为 true，同时本次新计算出来的结果作为缓存，用来与下次更新进行比较判断是否要 update。

OK，下面来看 sourceSelector 到底 select 了出个什么东西用来判断是否更新：

```
export default function finalPropsSelectorFactory(dispatch, {
  initMapStateToProps,
  initMapDispatchToProps,
  initMergeProps,
  ...options
}) {
  const mapStateToProps = initMapStateToProps(dispatch, options)
  const mapDispatchToProps = initMapDispatchToProps(dispatch, options)
  const mergeProps = initMergeProps(dispatch, options)

  if (process.env.NODE_ENV !== 'production') {
    verifySubselectors(mapStateToProps, mapDispatchToProps, mergeProps, options.displayName)
  }

  const selectorFactory = options.pure
    ? pureFinalPropsSelectorFactory
    : impureFinalPropsSelectorFactory

  return selectorFactory(
    mapStateToProps,
    mapDispatchToProps,
    mergeProps,
    dispatch,
    options
  )
}
```

函数中的 mapStateToProps 和 mapDispatchToProps 是通过两个 initxxxx 函数来生成的，options 中包含了用户传入的 mapXXXXToProps，我们拿 mapStateToProps 来说，在 connect 中

```
const initMapStateToProps = match(mapStateToProps, mapStateToPropsFactories, 'mapStateToProps')
```

match 的作用就是将第一个参数依次放入第二个参数的每一项中（第二个参数是一个数组），返回第一个不为 undefined 的结果。

正常情况下是返回这个函数的结果：

```
export function wrapMapToPropsFunc(mapToProps, methodName) {
  return function initProxySelector(dispatch, { displayName }) {
    const proxy = function mapToPropsProxy(stateOrDispatch, ownProps) {
      return proxy.dependsOnOwnProps
        ? proxy.mapToProps(stateOrDispatch, ownProps)
        : proxy.mapToProps(stateOrDispatch)
    }

    // allow detectFactoryAndVerify to get ownProps
    proxy.dependsOnOwnProps = true

    proxy.mapToProps = function detectFactoryAndVerify(stateOrDispatch, ownProps) {
      proxy.mapToProps = mapToProps
      proxy.dependsOnOwnProps = getDependsOnOwnProps(mapToProps)
      let props = proxy(stateOrDispatch, ownProps)

      if (typeof props === 'function') {
        proxy.mapToProps = props
        proxy.dependsOnOwnProps = getDependsOnOwnProps(props)
        props = proxy(stateOrDispatch, ownProps)
      }

      if (process.env.NODE_ENV !== 'production') 
        verifyPlainObject(props, displayName, methodName)

      return props
    }

    return proxy
  }
}
```
这个函数其实作用不大，返回一个 initProxy，proxy 其实还是调用了用户定义的 mapStateToProps，但是对初始化有作用。我们可以把它理解成一个健全版本的 mapStateToProps。回到 finalPropsSelectorFactory，我们拿到了一个初始化过的 mapStateToProps，mapDispatchToProps 和 mergeProps，等下，mergeProps是什么？这个函数可以看做是我们合并的策略，新的 props 或者 mapXXX 传入时，通过这个策略去合并出要传给包裹的组件的 props。

这三个参数会一并传入 pureFinalPropsSelectorFactory
```
export function pureFinalPropsSelectorFactory(
  mapStateToProps,
  mapDispatchToProps,
  mergeProps,
  dispatch,
  { areStatesEqual, areOwnPropsEqual, areStatePropsEqual }
) {
  let hasRunAtLeastOnce = false
  // 以下为缓存
  let state
  let ownProps
  let stateProps
  let dispatchProps
  let mergedProps

  // 界面初始化时的入口
  // 缓存 state, ownProps, stateProps, dispatchProps，mergedProps(同时也是我们要的结果)
  function handleFirstCall(firstState, firstOwnProps) {
    state = firstState
    ownProps = firstOwnProps
    stateProps = mapStateToProps(state, ownProps)
    dispatchProps = mapDispatchToProps(dispatch, ownProps)
    mergedProps = mergeProps(stateProps, dispatchProps, ownProps)
    hasRunAtLeastOnce = true
    return mergedProps
  }

  // 如果 store 和 props 都更新了
  function handleNewPropsAndNewState() {
    stateProps = mapStateToProps(state, ownProps)

    if (mapDispatchToProps.dependsOnOwnProps)
      dispatchProps = mapDispatchToProps(dispatch, ownProps)

    mergedProps = mergeProps(stateProps, dispatchProps, ownProps)
    return mergedProps
  }

  // 如果 props 更新了
  function handleNewProps() {
    if (mapStateToProps.dependsOnOwnProps)
      stateProps = mapStateToProps(state, ownProps)

    if (mapDispatchToProps.dependsOnOwnProps)
      dispatchProps = mapDispatchToProps(dispatch, ownProps)

    mergedProps = mergeProps(stateProps, dispatchProps, ownProps)
    return mergedProps
  }

  // 如果 store 更新了
  function handleNewState() {
    const nextStateProps = mapStateToProps(state, ownProps)
    const statePropsChanged = !areStatePropsEqual(nextStateProps, stateProps)
    stateProps = nextStateProps
    
    if (statePropsChanged)
      mergedProps = mergeProps(stateProps, dispatchProps, ownProps)

    return mergedProps
  }

  // 界面非初始化时的入口
  function handleSubsequentCalls(nextState, nextOwnProps) {
    const propsChanged = !areOwnPropsEqual(nextOwnProps, ownProps) // 被赋值为 shallowEqual
    const stateChanged = !areStatesEqual(nextState, state) // 被赋值为 shallowEqual
    state = nextState
    ownProps = nextOwnProps

    if (propsChanged && stateChanged) return handleNewPropsAndNewState() // props & store 都更新
    if (propsChanged) return handleNewProps() // props 更新
    if (stateChanged) return handleNewState() // store 更新
    return mergedProps // 如果 store 和 props 都没改变直接返回缓存
  }

  return function pureFinalPropsSelector(nextState, nextOwnProps) {
    return hasRunAtLeastOnce
      ? handleSubsequentCalls(nextState, nextOwnProps)
      : handleFirstCall(nextState, nextOwnProps)
  }
}
```
这个函数根据不同的更新来计算新的需要 merge 的状态（在 connect 中我们一般是不传入 mergeProps 这个参数的，会调用默认的 mergeProps 充当）。

然后传入 mergeProps 来进行合并，来看 mergeProps 的合并策略：
```
export function defaultMergeProps(stateProps, dispatchProps, ownProps) {
  return { ...ownProps, ...stateProps, ...dispatchProps }
}
```

就是用解构操作符返回一个新对象。再返回上面的 pureFinalPropsSelectorFactory，这个函数中缓存了 state, ownProps, stateProps, dispatchProps，mergedProps，如果 store 和 props 都不更新的话，那么直接返回上次计算的 mergedProps，如果 props 改变了，那么重新计算 props 的部分，其他的部分用之前的缓存，（store 更新同理，哪部分不变就用之前的缓存，变的再重新计算）再通过解构操作符返回一个浅拷贝的新对象用来表明已更新。

所以到这里就已经清楚了：**当 store 更新到来时，会调用 mergeProps 将新的参数直接 shallowMerge，重新将 ownProps， mapDispatchToProps，mapStateToProps 经过缓存优化（1. 不计算没改变的值 2. 如果都没改变那么直接返回缓存的结果，什么都不计算）。之后作为 selector 的结果传出去，再通过一次缓存进行比较（上层的函数也要再判断一次 selector 的结果变没变），如果不相同那么再更新被包裹的组件。**

### reselect
reselect 的作用就是缓存上一次 selector 的结果，和下一次 selector 的结果进行对比，如果相同就直接拿出之前缓存的 state。一般是为了防止其他无关状态的修改影响影响当前界面对应的 state，导致重渲染。在本例中，state 逻辑较简单，不存在其他的业务逻辑修改的情况。但是使用 reselect 优化的思路是一样，reselect 在 data -> view 之间添加了一层缓存来避免不必要的 re-render，我们可以使用它来进行 selector 的定义，很适合配合 react-redux 的 connect 之后生成的监听 store 的 HOC 使用。

