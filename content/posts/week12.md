---
date: 2019-02-27T16:00:30+08:00
tags: 
- 源码
- react
categories:
- 周记
title: React源码阅读笔记(0) - Reconciliation
---

_Joe按: React框架是现代前端必然会经历的一个划时代产物，与主打发布订阅者模式，小而美的Vue不同。React的编程思想实现得相对底层一些，没有各种神奇的语法糖和API限制。两个框架各有适用的场景和足够庞大的生态圈。作为一个生活在9102年的前端娱乐圈众，与其站队，不如都深入地学习一下其设计思想和源码。毕竟表达个人的偏好没有意义，有意义的是能深入到什么程度。_

React 16(Fiber版本)以前react的核心实现，大概分五个核心部分。

1. 建立虚拟DOM

2. 将虚拟DOM绘制成真实DOM

3. 由数据变化触发计算两个虚拟DOM树之间的差异，得到一个patchObject

4. 根据patchObject操作真实DOM

5. 事件代理

这一篇着重讨论第三部分，深入源码看看react是如何计算两个虚拟DOM树之间的差异。

要说看源码有什么技巧，大概就是从早期版本开始看会好入手很多，于是我翻看了react最早期的tag v0.3.3

### Reconcilation
找到两个状态之间的差异，生成一个最小变化的更新队列，这个过程在react的定义里称之为调和(Reconciliation)，也就是我们常说的diff算法的主要实施过程。

我们直接先找到调和部分的函数`updateMultiChild`，为了理解方便，我会稍微在注释里稍作注解：

[react源码](https://github.com/facebook/react/blob/v0.3.3/src/core/ReactMultiChild.js#L143)

```
/**
   * Reconciles new children with old children in three phases.
   * 更新分三步走，先增后删再移动
   * - Adds new content while updating existing children that should remain.
   * - Remove children that are no longer present in the next children.
   * - As a very last step, moves existing dom structures around.
   * - (Comment 1) `curChildrenDOMIndex` is the largest index of the current 
   *   rendered children that appears in the next children and did not need to
   *   be "moved". 就是说这个数表示新状态下不需要“移动”的子节点在旧状态中的index的最大值。听起来十分费解。
   *   但如comment 2 所说，这是判断元素是否需要移动位置的关键实现。
   *   如果一个子节点在新状态树中不需要被新增也不需要被删除，而它在旧状态树中的index小于comment 1中定义的`curChildrenDOMIndex`，那么它一定被移动了位置。
   * - (Comment 2) This is the key insight. If any non-removed child's previous
   *   index is less than `curChildrenDOMIndex` it must be moved.
   *
   * @param {?Object} children Flattened children object.
   */
  updateMultiChild: function(nextChildren, transaction) {
    if (!nextChildren && !this._renderedChildren) {
      return;
    } else if (nextChildren && !this._renderedChildren) {
      this._renderedChildren = {}; // lazily allocate backing store with nothing
    } else if (!nextChildren && this._renderedChildren) {
      nextChildren = {};
    }
    var rootDomIdDot = this._rootNodeID + '.';
    var markupBuffer = null;  // Accumulate adjacent new children markup. 这个变量用于将同一层级相邻的新元素通过缓存串在一起减少插入次数
    var numPendingInsert = 0; // How many root nodes are waiting in markupBuffer
    var loopDomIndex = 0;     // Index of loop through new children.
    var curChildrenDOMIndex = 0;  // See (Comment 1)
    /******************************
     *  这里开始比较所有的子节点  *
     ******************************/
    for (var name in nextChildren) {
      if (!nextChildren.hasOwnProperty(name)) {continue;}
      var curChild = this._renderedChildren[name];
      var nextChild = nextChildren[name];
      // 这个函数初步比较两个对象构造器是否相同，也就在比较是否是同类型元素
      if (shouldManageExisting(curChild, nextChild)) {
        // 如果碰到编辑的情况，先把新增的markup串都推入队列，重置markup缓存
        if (markupBuffer) {
          this.enqueueMarkupAt(markupBuffer, loopDomIndex - numPendingInsert);
          markupBuffer = null;
        }
        numPendingInsert = 0;
        // 
        if (curChild._domIndex < curChildrenDOMIndex) { // (Comment 2)
          this.enqueueMove(curChild._domIndex, loopDomIndex);
        }
        curChildrenDOMIndex = Math.max(curChild._domIndex, curChildrenDOMIndex);
        // 更新操作，receiveProps会触发更新
        !nextChild.props.isStatic &&
          curChild.receiveProps(nextChild.props, transaction);
        curChild._domIndex = loopDomIndex;
      } else {
        // 这里处理增删，先直接把旧元素推入unmount队列
        if (curChild) {               // !shouldUpdate && curChild => delete
          this.enqueueUnmountChildByName(name, curChild);
          curChildrenDOMIndex =
            Math.max(curChild._domIndex, curChildrenDOMIndex);
        }
        // 然后把新元素推入mount队列
        if (nextChild) {              // !shouldUpdate && nextChild => insert
          this._renderedChildren[name] = nextChild;
          // 这个markup指的就是跑完组件的render方法后得到的markup language字符串，故用'+'运算拼接在一起
          var nextMarkup =
            nextChild.mountComponent(rootDomIdDot + name, transaction);
          markupBuffer = markupBuffer ? markupBuffer + nextMarkup : nextMarkup;
          numPendingInsert++;
          nextChild._domIndex = loopDomIndex;
        }
      }
      // 根据是否有下一层，操作层级计数
      loopDomIndex = nextChild ? loopDomIndex + 1 : loopDomIndex;
    }
    if (markupBuffer) {
      this.enqueueMarkupAt(markupBuffer, loopDomIndex - numPendingInsert);
    }
    for (var childName in this._renderedChildren) { // from other direction
      if (!this._renderedChildren.hasOwnProperty(childName)) { continue; }
      var child = this._renderedChildren[childName];
      if (child && !nextChildren[childName]) {
        this.enqueueUnmountChildByName(childName, child);
      }
    }
    this.processChildDOMOperationsQueue();
  }
};
```

这里`curChildrenDOMIndex`可能即使看了源码也不是那么直观, 引用《深入React技术栈》的解读

>首先对新集合的节点进行循环遍历，for (name in nextChildren)，通过唯一 key 可以判断新老集合中是否存在相同的节点，if (prevChild === nextChild)，如果存在相同节点，则进行移动操作，但在移动前需要将当前节点在老集合中的位置与 lastIndex 进行比较，if (child._mountIndex < lastIndex)，则进行节点移动操作，否则不执行该操作。这是一种顺序优化手段，lastIndex 一直在更新，表示访问过的节点在老集合中最右的位置（即最大的位置），如果新集合中当前访问的节点比 lastIndex 大，说明当前访问节点在老集合中就比上一个节点位置靠后，则该节点不会影响其他节点的位置，因此不用添加到差异队列中，即不执行移动操作，只有当访问的节点比 lastIndex 小时，才需要进行移动操作。

意思是如果一个节点在旧集合中的位置已经在你之前进行判断的最后一个节点的后面，那么这个节点已经在被diff过的节点的后面了和之前的diff过的节点在顺序上就已经是正确的了，不需要移动了，反之的节点需要被移动。

[具体`receiveProps`如何触发更新看这里](https://github.com/facebook/react/blob/v0.3.3/src/core/ReactCompositeComponent.js#L490)

```
_receivePropsAndState: function(nextProps, nextState, transaction) {
    if (!this.shouldComponentUpdate ||
        this.shouldComponentUpdate(nextProps, nextState)) {
      // Will set `this.props` and `this.state`.
      this._performComponentUpdate(nextProps, nextState, transaction);
    } else {
      // If it's determined that a component should not update, we still want
      // to set props and state.
      this.props = nextProps;
      this.state = nextState;
    }
  },
```
更新组件状态的函数
```
/**
   * Updates the component's currently mounted DOM representation.
   *
   * By default, this implements React's rendering and reconciliation algorithm.
   * Sophisticated clients may wish to override this.
   *
   * @param {ReactReconcileTransaction} transaction
   * @internal
   * @overridable
   */
  updateComponent: function(transaction) {
    var currentComponent = this._renderedComponent;
    var nextComponent = this._renderValidatedComponent();
    if (currentComponent.constructor === nextComponent.constructor) {
      if (!nextComponent.props.isStatic) {
        currentComponent.receiveProps(nextComponent.props, transaction);
      }
    } else {
      // These two IDs are actually the same! But nothing should rely on that.
      var thisID = this._rootNodeID;
      var currentComponentID = currentComponent._rootNodeID;
      currentComponent.unmountComponent();
      var nextMarkup = nextComponent.mountComponent(thisID, transaction);
      ReactComponent.DOMIDOperations.dangerouslyReplaceNodeWithMarkupByID(
        currentComponentID,
        nextMarkup
      );
      this._renderedComponent = nextComponent;
    }
  },
```
然后转给了DOM操作模块进行更新操作
[更新DOM操作的相关代码在这里](https://github.com/facebook/react/blob/v0.3.3/src/domUtils/Danger.js)


### 那么react是如何触发diff比对的呢？

更新数据时react会建立一个transaction，顾名思义，用于保障更新的原子性、一致性、隔离性和持久性。通过调用`ReactComponent.ReactReconcileTransaction.getPooled()`来获得新的一个transaction，然后收集所有的状态更新，比对出diff，然后一次性更新到DOM中去。
在源码中搜索了这个方法的调用，一共有四处

|序号|模块、函数名|说明|
---|:--:|---:|
|1|[ReactComponent.js replaceProps](https://github.com/facebook/react/blob/v0.3.3/src/core/ReactComponent.js#L192)|用于已废弃的`ReactComponent.update()`和`ReactComponent.updateAll()`, 只在测试代码里找到了调用，还有在初始化renderComponent的时候如果有缓存，会用来从缓存中更新节点。|
|2|[ReactComponent.js mountComponentIntoNode](https://github.com/facebook/react/blob/v0.3.3/src/core/ReactComponent.js#L310)|顾名思义，mount新节点或首次渲染的时候会用。|
|3|[ReactCompositeComponent.js forceUpdate](https://github.com/facebook/react/blob/v0.3.3/src/core/ReactCompositeComponent.js#L672)|对外开放的API
|4|[ReactCompositeComponent.js replaceState](https://github.com/facebook/react/blob/v0.3.3/src/core/ReactCompositeComponent.js#L561)|不开放调用，只能通过另一个API触发，也就是我们最常用的setState，做一层封装是为了在setState的阶段收集各个setState提交上来的partialState进行合并。

- [ReactComponent.js的replaceProps方法](https://github.com/facebook/react/blob/v0.3.3/src/core/ReactComponent.js#L192) - 用于已废弃的`ReactComponent.update()`和`ReactComponent.updateAll()`, 只在测试代码里找到了调用，还有在初始化renderComponent的时候如果有缓存，会用来从缓存中更新节点。
- [ReactComponent.js的mountComponentIntoNode方法](https://github.com/facebook/react/blob/v0.3.3/src/core/ReactComponent.js#L310) - 顾名思义，mount新节点或首次渲染的时候会用。
- [ReactCompositeComponent.js中的forceUpdate](https://github.com/facebook/react/blob/v0.3.3/src/core/ReactCompositeComponent.js#L672) - 对外开放的API
- [ReactCompositeComponent.js中的replaceState方法)](https://github.com/facebook/react/blob/v0.3.3/src/core/ReactCompositeComponent.js#L561) - 不开放调用，只能通过另一个API触发，也就是我们最常用的setState，做一层封装是为了在setState的阶段收集各个setState提交上来的partialState进行合并。

### 所以我们常说key在Reconciliation中的角色是什么呢？

无论是react官网还是来自核心开发者[@vjeux](https://twitter.com/vjeux)关于[diff算法](https://calendar.perfplanet.com/2013/diff/)的知名文章，都强调key在管理组件生命周期中的重要性。那么为什么上述核心比对的函数中没有找到跟key相关的部分呢？

其实如果仔细看源码[这里](https://github.com/facebook/react/blob/v0.3.3/src/core/ReactMultiChild.js#L156)的话会发现`updateMultiChild`函数的参数的`children`和`this._renderedChildren`是key-value对的对象形式，而非直觉上的数组形式

```
// ...
for (var name in nextChildren) {
      if (!nextChildren.hasOwnProperty(name)) {continue;}
      var curChild = this._renderedChildren[name];
      var nextChild = nextChildren[name];
// ...
```
这个name就是根据key生成的，所以为什么key能够如我们一直熟识地帮助react的diff算法识别是否需要重渲？如果key不一样，就匹配不到`this._renderedChildren`的任何一个name，于是会被视作一个新插入的元素，不会进到更新dom的生命周期里，而是直接走mount新元素的流程。

当然，直接以对象形式存储children太过简单粗暴，而且语义不符合直觉，所以这里的存法只是为了缓存。react中有一个工具函数叫[`flattenChildren`](https://github.com/facebook/react/blob/v0.3.3/src/utils/flattenChildren.js)， 专门用来把子节点“拍平”了储存在对象中用于缓存比对。核心代码非常短。

```
/**
 * If there is only a single child, it still needs a name.
 */
var ONLY_CHILD_NAME = '0';

var flattenChildrenImpl = function(res, children, nameSoFar) {
  if (Array.isArray(children)) {
    for (var i = 0; i < children.length; i++) {
      flattenChildrenImpl(res, children[i], nameSoFar + '[' + i + ']');
    }
  } else {
    var type = typeof children;
    var isOnlyChild = nameSoFar === '';
    var storageName = isOnlyChild ? ONLY_CHILD_NAME : nameSoFar;
    if (children === null || children === undefined || type === 'boolean') {
      res[storageName] = null;
    } else if (children.mountComponentIntoNode) {
      /* We found a component instance */
      res[storageName] = children;
    } else {
      if (type === 'object') {
        throwIf(children && children.nodeType === 1, INVALID_CHILD);
        for (var key in children) {
          if (children.hasOwnProperty(key)) {
            flattenChildrenImpl(
              res,
              children[key],
              nameSoFar + '{' + escapeTextForBrowser(key) + '}'
            );
          }
        }
      } else if (type === 'string') {
        res[storageName] = new ReactTextComponent(children);
      } else if (type === 'number') {
        res[storageName] = new ReactTextComponent('' + children);
      }
    }
  }
};

/**
 * Flattens children that are typically specified as `props.children`.
 * @return {!Object} flattened children keyed by name.
 */
function flattenChildren(children) {
  if (children === null || children === undefined) {
    return children;
  }
  var result = {};
  flattenChildrenImpl(result, children, '');
  return result;
}

```

不过从key的缓存找上`flattenChildren`这个函数都是基于我的猜想，当我真正仔细看了一下这个函数发现被打脸了。。。

上述代码似乎完全按照index建立索引，没有key什么事，吓得我又在源码中找了一圈，确定没有key的相关实现。

怎么回事呢？于是我把tag切换到了后一个版本v0.4.0，发现这个函数出现了一点小小的变化

核心逻辑被抽到了另一个模块里，从模块名`traverseAllChildren`看出，是用来遍历子节点树的。
我们重新来看一下traverseAllChildren这个模块
[源码地址](https://github.com/facebook/react/blob/v0.4.0/src/utils/traverseAllChildren.js#L64)

```
/**
 * @param {?*} children Children tree container.
 * @param {!string} nameSoFar Name of the key path so far.
 * @param {!number} indexSoFar Number of children encountered until this point.
 * @param {!function} callback Callback to invoke with each child found.
 * @param {?*} traverseContext Used to pass information throughout the traversal
 * process.
 * @return {!number} The number of children in this subtree.
 */
var traverseAllChildrenImpl =
  function(children, nameSoFar, indexSoFar, callback, traverseContext) {
    var subtreeCount = 0;  // Count of children found in the current subtree.
    if (Array.isArray(children)) {
      for (var i = 0; i < children.length; i++) {
        var child = children[i];
        var nextName = nameSoFar + '[' + ReactComponent.getKey(child, i) + ']';
        var nextIndex = indexSoFar + subtreeCount;
        subtreeCount += traverseAllChildrenImpl(
          child,
          nextName,
          nextIndex,
          callback,
          traverseContext
        );
      }
    } else {
      var type = typeof children;
      var isOnlyChild = nameSoFar === '';
      // If it's the only child, treat the name as if it was wrapped in an array
      // so that it's consistent if the number of children grows
      var storageName = isOnlyChild ?
        '[' + ReactComponent.getKey(children, 0) + ']' :
        nameSoFar;
      if (children === null || children === undefined || type === 'boolean') {
        // All of the above are perceived as null.
        callback(traverseContext, null, storageName, indexSoFar);
        subtreeCount = 1;
      } else if (children.mountComponentIntoNode) {
        callback(traverseContext, children, storageName, indexSoFar);
        subtreeCount = 1;
      } else {
        if (type === 'object') {
          throwIf(children && children.nodeType === 1, INVALID_CHILD);
          for (var key in children) {
            if (children.hasOwnProperty(key)) {
              subtreeCount += traverseAllChildrenImpl(
                children[key],
                nameSoFar + '{' + key + '}',
                indexSoFar + subtreeCount,
                callback,
                traverseContext
              );
            }
          }
        } else if (type === 'string') {
          var normalizedText = new ReactTextComponent(children);
          callback(traverseContext, normalizedText, storageName, indexSoFar);
          subtreeCount += 1;
        } else if (type === 'number') {
          var normalizedNumber = new ReactTextComponent('' + children);
          callback(traverseContext, normalizedNumber, storageName, indexSoFar);
          subtreeCount += 1;
        }
      }
    }
    return subtreeCount;
  };

/**
 * Traverses children that are typically specified as `props.children`, but
 * might also be specified through attributes:
 *
 * - `traverseAllChildren(this.props.children, ...)`
 * - `traverseAllChildren(this.props.leftPanelChildren, ...)`
 *
 * The `traverseContext` is an optional argument that is passed through the
 * entire traversal. It can be used to store accumulations or anything else that
 * the callback might find relevant.
 *
 * @param {?*} children Children tree object.
 * @param {!function} callback To invoke upon traversing each child.
 * @param {?*} traverseContext Context for traversal.
 */
function traverseAllChildren(children, callback, traverseContext) {
  if (children !== null && children !== undefined) {
    traverseAllChildrenImpl(children, '', 0, callback, traverseContext);
  }
}
```
跟之前粗暴地用index做key相比，多了一个函数叫`ReactComponent.getKey`, 源码如下：

```
// @class ReactComponent
...
/**
   * Generate a key string that identifies a component within a set.
   *
   * @param {*} component A component that could contain a manual key.
   * @param {number} index Index that is used if a manual key is not provided.
   * @return {string}
   * @internal
   */
  getKey: function(component, index) {
    if (component && component.props && component.props.key != null) {
      // Explicit key
      return '' + component.props.key;
    }
    // Implicit key determined by the index in the set
    return '' + index;
  },
...
```
这里我们可以看到它的逻辑是有key取key，没key取index。

至于调用入口还是老地方, 看`flattenChildren`函数

```
/**
 * Flattens children that are typically specified as `props.children`.
 * @return {!object} flattened children keyed by name.
 */
function flattenChildren(children) {
  if (children === null || children === undefined) {
    return children;
  }
  var result = {};
  traverseAllChildren(children, flattenSingleChildIntoContext, result);
  return result;
}
```

从代码我们可以看出`flattenChildren`函数的几个特性：

1. 返回的`result`对象中包含了其全部子节点，每多一个层级，key的命名便会多一层[x]，key之间以层级隔离，这便是为什么react强调key只需要同级之间唯一不需要全局唯一。
2. 默认以children在自己那一个层级的index作为key，如果children本身为对象属性且配置了自己的key，就取这个key作为它在这个层级的key
3. key的设定在更新DOM的过程中扮演非常重要的角色，其原理就是基于哈希缓存的。key不变就是一个缓存中的旧元素，key变了就是一个新元素。这便是有些小技巧会建议你"通过改变组件的key值来强制刷新组件及其子组件生命周期"。
4. 引用react作者之一Paul O’Shannessy的话, "Key is not really about performance, it’s more about identity"


### 总结

看了以上源码，总结几个核心问题。

- React是如何将树的编辑算法复杂度从O(n^3)降到的O(n)的？
  1)只做层级间比对，不做跨层级比对。2)根据子节点构造器做简单初步判断，构造器都不一样直接更新。3)构造器相同的情况下根据子节点key做二次判断，key不一样直接更新。由此做到一次遍历所有子节点得到所有差异。

- React是如何触发渲染的？
  有mount新元素和对旧元素调用setState两种途径。react会基于事件循环在每一次loop的末尾收集全部并计算的更新，一次性更新到真实DOM中去

- 最后引申一个小小的思考，根据算法源码的解读，我们可以知道如果一个元素的父级元素被标记为需要rerender，那不管它本身有没有被rerender的需要，它都会被rerender。react为了能让你手动避免掉这个部分的rerender，提供了业内话题性极大的`shouldComponentUpdate`钩子，这都是后话了，在此给自己列个TODO，专门收集一下SCU相关的观点和当下各种解决方案组合。


官网文献链接：

[Reconciliation](<https://reactjs.org/docs/reconciliation.html>)

[SCU]([https://reactjs.org/docs/optimizing-performance.html\#shouldcomponentupdate-in-action](https://reactjs.org/docs/optimizing-performance.html#shouldcomponentupdate-in-action))
