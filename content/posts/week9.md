---
date: 2019-01-12T10:00:30+08:00
tags: 
- javascript
- koa2
categories:
- 周记
title: koa2 源码浅析
---
_Joe按：koa2源码浅析，真的浅。_

### 目录
koa2 的目录结构相当整洁，只有四个文件，搭起了整个 server 的框架，koa 只是负「开头」（接受请求）和「结尾」（响应请求），对请求的处理都是由中间件来实现。

```
├── lib
│   ├── application.js // 负责串联起 context request response 和中间件
│   ├── context.js // 一次请求的上下文
│   ├── request.js // koa 中的请求
│   └── response.js  // koa 中的响应
└── package.json
```

application.js：负责串联起 context request response 和中间件

context.js：一次请求的上下文

request.js：koa 中的请求

response.js：koa 中的响应

### 流程
我们从官网给的一个最小化的 demo 一步一步解析 koa 的源码了解其整个流程：

```
const Koa = require('koa');

// 初始化
const app = new Koa();

// 添加中间件
app.use(async ctx => {
  ctx.body = 'Hello World';
});

// 启动
app.listen(3000);
```

### 初始化
Koa 这个类是从 application.js 中导入的 Application，Application 继承自 Emmiter，Emmiter 唯一的作用就是去回调 onerror 事件，因为一次成功的 HTTP 请求会由 HTTP 模块来负责回调，但 error 是可能在中间件中 throw 出来的，此时就需要去通知 app 发生错误了。这在运行中的部分会有分析。

既然是初始化，就来看 koa 的构造函数：

```
  constructor() {
    super();

    this.proxy = false;
    this.middleware = []; // 存储中间件，构造洋葱模型
    this.subdomainOffset = 2;
    this.env = process.env.NODE_ENV || "development";
    this.context = Object.create(context); // 继承自 context
    this.request = Object.create(request); // 继承自 request
    this.response = Object.create(response); // 继承自 response
    if (util.inspect.custom) {
      this[util.inspect.custom] = this.inspect;
    }
  }
```
request 和 response 就是继承自 ./request.js 和 ./response， ./request.js 和 ./response 中的 request 和 response 做的事情并不复杂，就是将 Node 原生的 http 的 req 和 res 再次做了封装，便于读取和设置，每个属性和方法的封装都不复杂，具体的属性和方法 koa 的文档上已经写的很清楚了，这里就不再介绍了。

至于 context，context 是一个「传递的纽带」，每个中间件传递的就是 context，它承载了这次访问的所有内容，直到其被最终 response 掉。

其实看源码 context 就是一个普通的对象，普通对象就可以完成「记录并传递」这个作用，context 还额外提供了 error 和 cookie 的处理，因为 app 是继承自 `Emmiter` 只有它能订阅 `onerror` 事件，所以传递的 context 要包裹一个 app 的 `onerror` 事件来在中间件中传递。

### 运行中
运行中有一个很重要和基本的概念就是：HTTP 请求是幂等的。

> Methods can also have the property of "idempotence" in that (aside from error or expiration issues) the side-effects of N > 0 identical requests is the same as for a single request.

一次和多次请求某一个资源应该具有同样的副作用，也就是说每个 HTTP 请求都会有一套全新的 context，request，response。

整个运行中的流程如下：

![IMAGE](/pimg/E69724CA7532556250D8B7190D06991B.jpg)

核心代码如下：

构建新的 context，request，response，其中 context/request/response/app 中各种引用对方来保证各自方便的传递。

这里比较有意思的是，在构造函数中，this.context 是继承自 context.js 文件，这里在每次请求时又继承自 `this.context`。
这样的好处是你可以为你的 app 设定一个类似模板的 context，这样一来每个请求的 context 在继承时就会有一些预设的方法或属性，同理 `this.request` 和 `this.response`。

```
  createContext(req, res) {
    const context = Object.create(this.context); // 从 this.context 中继承
    const request = (context.request = Object.create(this.request)); // 从 this.request 中继承
    const response = (context.response = Object.create(this.response)); // this.response 中继承
    context.app = request.app = response.app = this;
    context.req = request.req = response.req = req;
    context.res = request.res = response.res = res;
    request.ctx = response.ctx = context;
    request.response = response;
    response.request = request;
    context.originalUrl = request.originalUrl = req.url;
    // 中间件中用来传递的命名空间
    context.state = {};
    return context;
  }
```
调用 http 的端口监听
```
  listen(...args) {
    debug("listen");
    // node 的 http 模块会去监听这个端口并触发回调
    const server = http.createServer(this.callback());
    return server.listen(...args);
  }
```
构建 http 的回调函数
```
  callback() {
    // compose 一个中间件洋葱模型【运行时确定中间件】
    const fn = compose(this.middleware);

    // koa 会挂载一个默认的错误处理【运行时确定异常处理】
    if (!this.listenerCount("error")) this.on("error", this.onerror);

    // node 的 http 模块会回调这个函数
    const handleRequest = (req, res) => {
      // 构造 koa context【运行时构造新的 ctx req res】
      const ctx = this.createContext(req, res);
      // 由 ctx 和中间件去响应
      return this.handleRequest(ctx, fn);
    };
    return handleRequest;
  }
```
中间件及异常处理
```
  handleRequest(ctx, fnMiddleware) {
    const res = ctx.res;
    res.statusCode = 404;
    const onerror = err => ctx.onerror(err);
    // koa 在中间件处理完之后返回 response
    const handleResponse = () => respond(ctx);
    // Execute a callback when a HTTP request closes, finishes, or errors.
    // 兜底的错误处理
    onFinished(res, onerror);

    return fnMiddleware(ctx) // 中间件先处理
      .then(handleResponse) // ctx.res 就是最后想要的结果
      .catch(onerror); // catch 中间件中抛出的异常
  }
```
koa 自带的成功 res 处理
```
function respond(ctx) {
  // allow bypassing koa
  if (false === ctx.respond) return;

  if (!ctx.writable) return;

  const res = ctx.res;
  let body = ctx.body;
  const code = ctx.status;

  // ignore body
  // 后面都是处理 body 的了，如果是不需要 body 的请求，则 body 直接为 null 就可以返回了。
  // statuses 库中有三种请求会直接是无 body 的（其实也就是根据 HTTP 规范），分别是：
  // 204: No Content 响应，就表示执行成功，但是没有数据，浏览器不用刷新页面，也不用导向新的页面。
  // 205: Reset Content 响应，205的意思是在接受了浏览器 POST 请求以后处理成功以后，告诉浏览器，执行成功了，请清空用户填写的 Form 表单，方便用户再次填写。
  // 304: 协商缓存
  // refer: http://www.laruence.com/2011/01/20/1844.html

  if (statuses.empty[code]) {
    // strip headers
    ctx.body = null;
    return res.end();
  }

  // HEAD 请求:
  if ("HEAD" == ctx.method) {
    if (!res.headersSent && isJSON(body)) {
      ctx.length = Buffer.byteLength(JSON.stringify(body));
    }
    return res.end();
  }

  // status body
  // 表示状态码的 body:
  if (null == body) {
    if (ctx.req.httpVersionMajor >= 2) {
      body = String(code);
    } else {
      body = ctx.message || String(code);
    }
    if (!res.headersSent) {
      ctx.type = "text";
      ctx.length = Buffer.byteLength(body);
    }
    return res.end(body);
  }

  // responses
  if (Buffer.isBuffer(body)) return res.end(body); // 允许返回 Buffer 类型
  if ("string" == typeof body) return res.end(body); // 允许返回 string 类型
  if (body instanceof Stream) return body.pipe(res); // 允许返回 stream 类型

  // body: json
  // 允许返回 json 类型，还是会被 JSON.stringify 化为字符串
  body = JSON.stringify(body);
  if (!res.headersSent) {
    ctx.length = Buffer.byteLength(body);
  }
  res.end(body);
}
```

在初始化中提到了，koa 继承自 Emiiter，是为了处理可能在任意时间抛出的异常所以订阅了 error 事件。error 处理有两个层面，一个是 app 层面全局的（主要负责 log），另一个是一次响应过程中的 error 处理（主要决定响应的结果），koa 有一个默认 app-level 的 onerror 事件，用来输出错误日志。
```
  onerror(err) {
    if (!(err instanceof Error))
      throw new TypeError(util.format("non-error thrown: %j", err));

    if (404 == err.status || err.expose) return;
    if (this.silent) return;

    const msg = err.stack || err.toString();
    console.error();
    console.error(msg.replace(/^/gm, "  "));
    console.error();
  }
```