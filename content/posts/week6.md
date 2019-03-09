---
date: 2018-12-16T15:20:30+08:00
tags: 
- react
- react-router-4
- 源码
categories:
- 周记
title: React router 4源码浅析
---
_Joe按: 本文将从各类路由的实现原理入手，对 react-router的源码进行分析，来理解它是如何帮助我们管理路由状态的。_

### 路由
在分析源码之前，先来对路由有一个认识。在SPA盛行之前，还不存在前端层面的路由概念，每个URL对应一个页面，所有的跳转或者链接都通过 <a> 标签来完成，随着SPA的逐渐兴盛及HTML5的普及，hash路由及基于history的路由库越来越多。

路由库最大的作用就是同步 URL 与其对应的回调函数。对于基于`history`的路由，它通过`history.pushState`来修改 URL，通过`window.addEventListener('popstate', callback)`来监听前进/后退事件；对于hash路由，通过操作`window.location`字符串来更改hash，通过`window.addEventListener('hashchange', callback) `来监听URL的变化。

### SPA 路由实现

#### hash 路由
```
class Router {
  constructor() {
    // 储存 hash 与 callback 键值对
    this.routes = {};
    // 当前 hash
    this.currentUrl = '';
    // 记录出现过的 hash
    this.history = [];
    // 作为指针,默认指向 this.history 的末尾,根据后退前进指向 history 中不同的 hash
    this.currentIndex = this.history.length - 1;
    this.backIndex = this.history.length - 1
    this.refresh = this.refresh.bind(this);
    this.backOff = this.backOff.bind(this);
    // 默认不是后退操作
    this.isBack = false;
    window.addEventListener('load', this.refresh, false);
    window.addEventListener('hashchange', this.refresh, false);
  }

  route(path, callback) {
    this.routes[path] = callback || function() {};
  }

  refresh() {
    console.log('refresh')
    this.currentUrl = location.hash.slice(1) || '/';
    this.history.push(this.currentUrl);
    this.currentIndex++;
    if (!this.isBack) {
      this.backIndex = this.currentIndex
    }
    this.routes[this.currentUrl]();
    console.log('指针:', this.currentIndex, 'history:', this.history);
    this.isBack = false;
  }
  // 后退功能
  backOff() {
    // 后退操作设置为true
    console.log(this.currentIndex)
    console.log(this.backIndex)
    this.isBack = true;
    this.backIndex <= 0 ?
      (this.backIndex = 0) :
      (this.backIndex = this.backIndex - 1);
    location.hash = `#${this.history[this.backIndex]}`;
  }
}
```

完整实现参考[hash router](https://juejin.im/post/5ac61da66fb9a028c71eae1b)

其实知道了路由的原理，想要实现一个hash 路由并不困难，比较需要注意的是`backOff`的实现，包括hash router中对 `backOff`的实现也是有bug的，浏览器的回退会触发`hashChange` 所以会在`history`中 push 一个新的路径，也就是每一步都将被记录。所以需要一个`backIndex` 来作为返回的index的标识，在点击新的URL的时候再将`backIndex`回归为 `this.currentIndex`。

### 基于 history 的路由实现
```
class Routers {
  constructor() {
    this.routes = {};
    // 在初始化时监听popstate事件
    this._bindPopState();
  }
  // 初始化路由
  init(path) {
    history.replaceState({path: path}, null, path);
    this.routes[path] && this.routes[path]();
  }
  // 将路径和对应回调函数加入hashMap储存
  route(path, callback) {
    this.routes[path] = callback || function() {};
  }

  // 触发路由对应回调
  go(path) {
    history.pushState({path: path}, null, path);
    this.routes[path] && this.routes[path]();
  }
  // 后退
  backOff(){
    history.back()
  }
  // 监听popstate事件
  _bindPopState() {
    window.addEventListener('popstate', e => {
      const path = e.state && e.state.path;
      this.routes[path] && this.routes[path]();
    });
  }
}
```

相比 hash 路由，h5 路由不再需要有些丑陋去的去修改 `window.location` 了，取而代之使用 `history.pushState` 来完成对 `window.location` 的操作，使用`window.addEventListener('popstate', callback)` 来对前进/后退进行监听，至于后退则可以直接使用`window.history.back()` 或者`window.history.go(-1)`来直接实现，由于浏览器的`history` 控制了前进/后退的逻辑，所以实现简单了很多。

### react 中的路由
react作为一个前端视图库，本身是不具有除了view（数据与界面之间的抽象）之外的任何功能的，为react引入一个路由库的目的与上面的普通 SPA目的一致，只不过上面路由更改触发的回调函数是我们自己写的操作 DOM的函数；在react中我们不直接操作DOM，而是管理抽象出来的 VDOM或者说JSX，对react的来说路由需要管理组件的生命周期，对不同的路由渲染不同的组件。

react-router4依赖[history](https://github.com/ReactTraining/history)库作为`window.history`的加强库

#### match
源自history库，表示当前的URL与`path`的匹配的结果
```
match: {
    path: "/", // 用来匹配的 path
	url: "/", // 当前的 URL
	params: {}, // 路径中的参数
	isExact: pathname === "/" // 是否为严格匹配
}
```

#### location
还是源自history库，是history库基于`window.location`的一个衍生。
```
hash: "" // hash
key: "nyi4ea" // 一个 uuid
pathname: "/explore" // URL 中路径部分
search: "" // URL 参数
state: undefined // 路由跳转时传递的 state
```

#### packages
react-router4将路由拆成了几个包
- react-router 负责通用的路由逻辑
- react-router-dom 负责浏览器的路由管理
- react-router-native 负责 react-native 的路由管理，通用的部分直接从 react-router 中导入，用户只需引入 react-router-dom 或 react-router-native 即可，react-router 作为依赖存在不再需要单独引入。

#### Router
React支持多种Router，我们先拿`BrowserRouter`举例。

BrowserRouter 的源码在react-router-dom中，它是一个高阶组件，在内部创建一个全局的`history`对象（可以监听整个路由的变化），并将`history`作为props传递给react-router的Router组件（Router组件再会将这个history的属性作为`context`传递给子组件）

整个Router的核心是在react-router的Router组件中，如下，借助`context`向Route传递组件，这也解释了为什么Router要在所有Route的外面。
```
 getChildContext() {
    return {
      router: {
        ...this.context.router,
        history: this.props.history,
        route: {
          location: this.props.history.location,
          match: this.state.match
        }
      }
    };
  }
```
这是Router传递给子组件的context，事实上Route也会将`router作为`context`向下传递，如果我们在Route渲染的组件中加入
```
  static contextTypes = {
    router: PropTypes.shape({
      history: PropTypes.object.isRequired,
      route: PropTypes.object.isRequired,
      staticContext: PropTypes.object
    })
  };
```
来通过context访问router，不过react-router4一般通过props传递，将history, location, match作为三个独立的props传递给要渲染的组件，这样访问起来方便一点（实际上已经完全将router对象的属性完全传递了）。

在Router的`componentWillMount`中， 添加了
```
  componentWillMount() {
    const { children, history } = this.props;

    invariant(
      children == null || React.Children.count(children) === 1,
      "A <Router> may have only one child element"
    );

    // Do this here so we can setState when a <Redirect> changes the
    // location in componentWillMount. This happens e.g. when doing
    // server rendering using a <sStaticRouter>.
    this.unlisten = history.listen(() => {
      this.setState({
        match: this.computeMatch(history.location.pathname)
      });
    });
  }
```
`history.listen`能够监听路由的变化并执行回调事件。

在这里每次路由的变化执行的回调事件为
```
this.setState({
    match: this.computeMatch(history.location.pathname)
});
```
相比于在`setState`里做的操作，`setState`本身的意义更大 —— 每次路由变化 -> 触发顶层Router的回调事件 -> Router进行`setState` -> 向下传递`nextContext`（`context`中含有最新的`location`）-> 下面的Route获取新的`nextContext`判断是否进行渲染。

之所以把这个`subscribe`的函数写在`componentWillMount`里，就像源码中给出的注释：是为了SSR的时候，能够使用Redirect。

### Route
Route的作用是匹配路由，并传递给要渲染的组件props。

在Route的`componentWillReceiveProps`中
```
  componentWillReceiveProps(nextProps, nextContext) {
    ...
    this.setState({
      match: this.computeMatch(nextProps, nextContext.router)
    });
  }
```
Route接受上层的Router传入的`context`，Router中的`history` 监听着整个页面的路由变化，当页面发生跳转时，`history` 触发监听事件，Router 向下传递`nextContext`，就会更新Route的`props`和`context`来判断当前 Route的`path`是否匹配`location`，如果匹配则渲染，否则不渲染。

是否匹配的依据就是`computeMatch` 这个函数，在下文会有分析，这里只需要知道匹配失败则`match`为 `null`，如果匹配成功则将`match`的结果作为`props`的一部分，在`render` 中传递给传进来的要渲染的组件。

接下来看一下Route的`render`部分。
```
  render() {
    const { match } = this.state; // 布尔值，表示 location 是否匹配当前 Route 的 path
    const { children, component, render } = this.props; // Route 提供的三种可选的渲染方式
    const { history, route, staticContext } = this.context.router; // Router 传入的 context
    const location = this.props.location || route.location;
    const props = { match, location, history, staticContext };

    if (component) return match ? React.createElement(component, props) : null; // Component 创建

    if (render) return match ? render(props) : null; // render 创建

    if (typeof children === "function") return children(props); // 回调 children 创建

    if (children && !isEmptyChildren(children)) // 普通 children 创建
      return React.Children.only(children);

    return null;
  }
```
react-routerr4提供了三种渲染组件的方法：`component` props，`render` props和`children` props，渲染的优先级也是依次按照顺序，如果前面的已经渲染后了，将会直接return。

- component (props) —— 由于使用 React.createElement 创建，所以可以传入一个 class component。
- render (props) —— 直接调用 render() 展开子元素，所以需要传入 stateless function component。
- children (props) —— 其实和 render 差不多，区别是不判断 match，总是会被渲染。
- children（子元素）—— 如果以上都没有，那么会默认渲染子元素，但是只能有一个子元素。
这里解释一下官网的 tips，component 是使用 React.createElement 来创建新的元素，所以如果传入一个内联函数，比如
```
<Route path='/' component={()=>(<div>hello world</div>)}
```
的话，由于每次的`props.component`都是新创建的，所以React在diff 的时候会认为进来了一个全新的组件，所以会将旧的组件unmount，再 re-mount。这时候就要使用`render`，少了一层包裹的component 元素，`render`展开后的元素类型每次都是一样的，就不会发生re-mount 了（children也不会发生re-mount）。

### Switch
我们紧接着Route来看Switch，Switch是用来嵌套在Route的外面，当 Switch中的第一个Route匹配之后就不会再渲染其他的Route了。
```
  render() {
    const { route } = this.context.router;
    const { children } = this.props;
    const location = this.props.location || route.location;

    let match, child;
    React.Children.forEach(children, element => {
      if (match == null && React.isValidElement(element)) {
        const {
          path: pathProp,
          exact,
          strict,
          sensitive,
          from
        } = element.props;
        const path = pathProp || from;

        child = element;
        match = matchPath(
          location.pathname,
          { path, exact, strict, sensitive },
          route.match
        );
      }
    });

    return match
      ? React.cloneElement(child, { location, computedMatch: match })
      : null;
  }
```
Switch也是通过`matchPath`这个函数来判断是否匹配成功，一直按照Switch中`children`的顺序依次遍历子元素，如果匹配失败则`match`为`null`，如果匹配成功则标记这个子元素和它对应的`location`、`computedMatch`。在最后的时候使用`React.cloneElement`渲染，如果没有匹配到的子元素则返回`null`。

接下来我们看下`matchPath`是如何判断`location`是否符合`path`的。

### matchPath
matchPath返回的是一个如下结构的对象
```
  {
    path, // 用来进行匹配的路径，其实是直接导出的传入 matchPath 的 options 中的 path
    url: path === "/" && url === "" ? "/" : url, // 整个的 URL
    isExact, // url 与 path 是否是 exact 的匹配
    // 返回的是一个键值对的映射
    // 比如你的 path 是 /users/:id，然后匹配的 pathname 是 /user/123
    // 那么 params 的返回值就是 {id: '123'}
    params: keys.reduce((memo, key, index) => {
      memo[key.name] = values[index];
      return memo;
    }, {}) 
  }
```
这些信息将作为匹配的参数传递给Route和Switch（Switch 只是一个代理，它的作用还是渲染Route，Switch计算得到的`computedMatch`会传递给要渲染的Route，此时Route将直接使用这个`computedMatch`而不需要再自己来计算）。

在`matchPath`内部`compilePath`时，有个
```
const patternCache = {};
const cacheLimit = 10000;
let cacheCount = 0;
```
作为`pathToRegexp`的缓存，因为ES6的`import` 模块导出的是值的引用，所以将`patternCache`可以理解为一个全局变量缓存，缓存以`{option:{pattern: }}`的形式存储，之后如果需要匹配相同`pattern`和`option`的`path`，则可以直接从缓存中获得正则表达式和keys。

加缓存的原因是路由页面大部分情况下都是相似的，比如要访问`/user/123`或`/users/234`，都会使用`/user/:id`这个path去匹配，没有必要每次都生成一个新的正则表达式。SPA在页面整个访问的过程中都维护着这份缓存。

### Link
实际上我们可能写的最多的就是Link这个标签了，我们从它的`render`函数开始看
```
  render() {
    const { replace, to, innerRef, ...props } = this.props; // eslint-disable-line no-unused-vars

    invariant(
      this.context.router,
      "You should not use <Link> outside a <Router>"
    );

    invariant(to !== undefined, 'You must specify the "to" property');

    const { history } = this.context.router;
    const location =
      typeof to === "string"
        ? createLocation(to, null, null, history.location)
        : to;

    const href = history.createHref(location);
    // 最终创建的是一个 a 标签
    return (
      <a {...props} onClick={this.handleClick} href={href} ref={innerRef} />
    );
  }
```

可以看到 Link 最终还是创建一个a标签来包裹住要跳转的元素，但是如果只是一个普通的带`href`的a标签，那么就会直接跳转到一个新的页面而不是 SPA了，所以在这个a标签的`handleClick`中会`preventDefault`禁止默认的跳转，所以这里的`href`并没有实际的作用，但仍然可以标示出要跳转到的页面的URL并且有更好的html语义。

在`handleClick`中，对没有被 “preventDefault的 && 鼠标左键点击的 && 非 _blank 跳转 的&& 没有按住其他功能键的“ 单击进行`preventDefault`，然后push进`history`中，这也是前面讲过的 —— 路由的变化与页面的跳转是不互相关联的，react-router4在`Link`中通过 history库的push调用了HTML5 history 的`pushState`，但是这仅仅会让路由变化，其他什么都没有改变。还记不记得 Router中的`listen`，它会监听路由的变化，然后通过`context`更新 `props`和`nextContext`让下层的Route去重新匹配，完成需要渲染部分的更新。

```
  handleClick = event => {
    if (this.props.onClick) this.props.onClick(event);

    if (
      !event.defaultPrevented && // onClick prevented default
      event.button === 0 && // ignore everything but left clicks
      !this.props.target && // let browser handle "target=_blank" etc.
      !isModifiedEvent(event) // ignore clicks with modifier keys
    ) {
      event.preventDefault();

      const { history } = this.context.router;
      const { replace, to } = this.props;

      if (replace) {
        history.replace(to);
      } else {
        history.push(to);
      }
    }
  };
```
withRouter

```
const withRouter = Component => {
  const C = props => {
    const { wrappedComponentRef, ...remainingProps } = props;
    return (
      <Route
        children={routeComponentProps => (
          <Component
            {...remainingProps}
            {...routeComponentProps}
            ref={wrappedComponentRef}
          />
        )}
      />
    );
  };

  C.displayName = `withRouter(${Component.displayName || Component.name})`;
  C.WrappedComponent = Component;
  C.propTypes = {
    wrappedComponentRef: PropTypes.func
  };

  return hoistStatics(C, Component);
};

export default withRouter;
```
withRouter的作用是让我们在普通的非直接嵌套在Route中的组件也能获得路由的信息，这时候我们就要`WithRouter(wrappedComponent)`来创建一个HOC 传递props，WithRouter的其实就是用Route包裹了SomeComponent的一个 HOC。

创建 Route 有三种方法，这里直接采用了传递`children`props的方法，因为这个 HOC 要原封不动的渲染`wrappedComponent`（`children` props比较少用得到，某种程度上是一个内部方法）。

在最后返回HOC时，使用了`hoistStatics`这个方法，这个方法的作用是保留SomeComponent类的静态方法，因为HOC是在`wrappedComponent`的外层又包了一层Route，所以要将`wrappedComponent`类的静态方法转移给新的Route，具体参见[Static Methods Must Be Copied Over](https://reactjs.org/docs/higher-order-components.html#static-methods-must-be-copied-over)。

### 理解
现在回到一开始的问题，重新理解一下点击一个 Link 跳转的过程。

有两件事需要完成：

1. 路由的改变
2. 页面的渲染部分的改变

过程如下：

1. 在最一开始`mount` Router的时候，Router在`componentWillMount` 中`listen`了一个回调函数，由`history`库管理，路由每次改变的时候触发这个回调函数。这个回调函数会触发 `setState`。
2. 当点击Link标签的时候，实际上点击的是页面上渲染出来的a标签，然后通过`preventDefault`阻止a标签的页面跳转。
3. Link 中也能拿到 Router -> Route中通过`context`传递的 `history`，执行`hitsory.push(to)`，这个函数实际上就是包装了一下`window.history.pushState()`，是HTML5 history的API，但是`pushState`之后除了地址栏有变化其他没有任何影响，到这一步已经完成了目标1：路由的改变。
4. 第1步中，路由改变是会触发Router的`setState`的，在Router那章有写道：每次路由变化 -> 触发顶层Router的监听事件 -> Router触发`setState` -> 向下传递新的`nextContext`（`nextContext`中含有最新的`location`）
5. 下层的Route拿到新的`nextContext`通过`matchPath`函数来判断`path`是否与`location`匹配，如果匹配则渲染，不匹配则不渲染，完成目标2：页面的渲染部分的改变。
