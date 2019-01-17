---
date: 2019-01-17T14:29:30+08:00
tags: 
- javascript
- 性能
categories:
- 周记
- 翻译
title: 符合CSP策略的非阻塞式script脚本加载器
---

_Joe按: 翻译自[A CSP Compliant non-blocking script loader](https://calendar.perfplanet.com/2018/a-csp-compliant-non-blocking-script-loader/), 原著[Philip Tellis](https://twitter.com/bluesmoon) & [Allan Wirth](https://twitter.com/Allan_Wirth)_

在2012年的性能日报中，我发布了我们在LogNormal/SOASTA项目中研发的[非阻塞script加载模式](https://calendar.perfplanet.com/2012/the-non-blocking-script-loader-pattern/)，该技术用于在关键渲染路径(Critical Rendering Path)外加载第三方javascript并确保不会对客户页面造成单点故障(SPoF)。

非阻塞script加载模式不像script的async和defer属性那样仍旧对script的onload事件造成阻塞，它允许javascript脚本完全不按顺序加载，永不阻塞，甚至允许script从未加载。
自2012年以来发生了很多变化。浏览器现在有新的方法在不阻塞页面的情况下加载script，而像CSP（Content Security Policy内容安全策略）这样的安全标准，以及一些XSS和CSRF检查器使得JavaScript中iframe的使用变得困难重重。因为我们在iframe内部使用的`document.write`方法也存在问题，因为Chrome现在会上报并认为即使在匿名iframe中使用也是不安全的。

我们还注意到，在JavaScript中创建一个iframe会给页面带来约80毫秒的延迟。对于通常在1秒内彻底加载完毕的网站（是的，我们已经使用了其中一些），这10％的增加至关重要。最后，尽管我们已经多次证实这不是真的，但还是会从网站方那里听说在`document`的HEAD部分中使用iframe会导致SEO问题。事实证明比起解释旧方案的复杂性，研发新的解决方案来得更容易一些。
所以，今天我们将讨论一个新的加载方案，旧方案的问题它一个也不存在，并且还能与CSP良好兼容。唯一的缺陷是不兼容IE6，好吧，不是什么太大的缺点。
	
### 预加载提示
我们将使用的第一个新标准就是预加载提示。具体来说，链接rel =“preload”as =“script”方法而不是HTTP标头。根据caniuse的说法，今天75％的浏览器支持这一点。 IE和Firefox是两个值得注意的例外。当然，仅仅将链接元素粘贴到文档的头部是不够的。我们需要附加加载和错误处理程序，并且如果不支持preload，也可以回退到其他方法，因此我们需要使用JavaScript来注入元素： 

```
var l = document.createElement("link");
if (l.relList && typeof l.relList.supports === "function" && l.relList.supports("preload") && ("as" in l)) {
    l.href = "";
    l.rel  = "preload";
    l.as   = "script";
    l.addEventListener("load", promote);
    l.addEventListener("error", fallback_loader);
    setTimeout(function() {
        if (!promoted) {
            fallback_loader();
        }
    }, LOADER_TIMEOUT);
    where.parentNode.appendChild(l);
}
else {
    fallback_loader();
}
```
预加载提示可以指定（支持的）浏览器异步下载资源，不阻塞其它事件，并将资源存储在script缓存中，以备script节点在可预见的将来引用。达到完全非阻塞的技巧在于，让script节点的注入仅发生在确定资源已被缓存之后，也就是预加载节点`load`事件触发的时候。这也是我们无法使用HTTP头的原因，因为你无法将JavaScript的处理方法加到HTTP头上。

上面代码中有一些未定义的项，大多数都不太重要。 `fallback_loader`是我们在预加载提示不可用时使用的方法，它可能只是旧的iframe加载器。`where`指向一个DOM节点的引用，表示在哪里注入link元素。它可能只是通过`document.currentScript`访问的当前script节点。

`promote`是一个函数，一旦确认文件已缓存，我们将用它来修改script的链接地址。`promoted`是一个布尔值，以确保完成后的停止。我们现在来看这两项的实现。

```
function promote() {
    var s;
    s = document.createElement("script");
    s.id = "a-unique-id-for-your-script";
    s.src = this.href;
    // 这里其实并不需要指定async属性，因为动态添加的script默认是异步的
    // 执行到这里的时候script已经被缓存，但仍有一些单纯的解析器
    // 会因为缺少async属性而认为它不是异步的，所以加上设置async属性为true
    s.async = true;
    this.parentNode.appendChild(s);
    promoted = true;
}
```

这里没什么太复杂的。 它的要点是s.src = this.href这行。由于`promote()`作为`link`元素的`load`事件处理方法执行，`this`指向link元素自身，`this.href`正是我们想要的url。 这真的很酷，因为这意味着我们可以用同一个处理方法加到不同的`link`元素上来获取他们的script。 我们也可以从`link`元素继承`id`。

在这个函数的最后我们将`promoted`设为true，我们还将在`fallback_loader()`中检查它的值，以避免fallback的重复执行。

### IFrame回落方案

在浏览器不支持或预加载失败的情况下，script的加载将回落到先前我们使用过的[非阻塞script加载模式](https://calendar.perfplanet.com/2012/the-non-blocking-script-loader-pattern/)的一个改进版。
```
    var s, iframe = document.createElement("iframe");

    iframe.src = "about:blank";
    
    iframe.title = "";
    iframe.role = "presentation";
    
    s = (iframe.frameElement || iframe).style;
    s.width = 0; s.height = 0; s.border = 0; s.display = "none";
    
    where.parentNode.insertBefore(iframe, where);
```
此段代码创建一个空的iframe，并将其注入`document`中的第一个`script`节点。 它将`title`设置为空白，将ARIA`role`属性设置为`presentation`，这样就能[告诉无障碍富网络应用辅助技术它是可以忽略的](https://www.w3.org/TR/wai-aria-practices-1.1/#presentation_role)。 也可以通过CSS从视觉展示上隐藏frame元素来实现相似的效果。

`document.domain`和IE
一旦我们有了`iframe`元素，父级便需要能够修改iframe DOM的内容。 为此，需要在子元素的`document`上调用`document.open`。

```
    try {
      win = iframe.contentWindow;
      doc = win.document.open();
    }
    catch (e) {
      // 如果再老版本IE上document.domain被改动，我们会拿到拒绝访问(access denied)的错误
      // 备注：有这个问题的浏览器也不支持内容安全策略CSP属性
      // 获取父级窗口的document.domain
      dom = document.domain;
      // 将iframe的src属性设为一个javascript脚本的url地址会立刻将它的document.domain设为和父级窗口一样
      // 这个特性让我们能够长时间访问iframe的document以注入我们的script脚本
      // 我们的script脚本后续 可能会需要做更多域名相关的适配
      iframe.src = "javascript:var d=document.open();d.domain='" + dom + "';void(0);";

      win = iframe.contentWindow;
      doc = win.document.open();
    }
```

在IE8中，如果父页面中已设置document.domain，则会被错误地拒绝。 要解决这个问题，加载方法会捕获安全性异常，并使用javascript uri的方法在子页面中显式设置document.domain。

在IE和Edge的所有版本中，如果父页面的`document.domain`后续发生变更，父页面便失去对子页面的访问权限。 要解决这个问题，我们的加载脚本第一件事就是要抓取`window.parent`并将其存储在本地。 如果无法访问`window.parent`，尝试遍历域名树的子域名直到命中与父节点匹配的域名。 如果加载方法在`document.domain`被页面上其他脚本更改之前执行，这样做可以缓解竞争的条件。

### 加载时触发

非阻塞iframe模式最重要的技巧在于等iframe自己的`load`事件触发以后再去加载script。这也是因为父页面的`onload`事件只会等iframe的`onload`事件加载完再触发，而不会等iframe`onload`完里面的资源全部加载后再触发。为此，iframe子页面的`load`事件需要绑定一个事件处理函数来实现。

```
  var bootstrap = function() {
    var js = doc.createElement("script");
    js.src = "";
    js.id = "a-unique-id-for-your-iframed-script";
    doc.body.appendChild(js);
  };

  try {
    win._l = bootstrap;

    if (win.addEventListener) {
      win.addEventListener("load", win._l, false);
    }
    else if (win.attachEvent) {
      win.attachEvent("onload", win._l);
    }
  }
  catch(e) {
    // 兼容IE8的不安全版本
    // 如果document.domain被修改，则不能用win，但我们可以用doc
    doc._l = function() {
      if (dom) {
        this.domain = dom;
      }
      bootstrap();
    }
    doc.write('<bo' + 'dy onload="document._l();">');
  }
  doc.close();

```

这段`bootstrap`方法在子页面中创建一个script元素来加载脚本，里面包含一个魔法ID，用来表达它已被iframe的加载器加载完毕。

然后，该脚本尝试在子页面的`window`上监听事件。在现代浏览器中，可以使用win.addEventListener执行此操作，在IE8中使用win.attachEvent。当document.domain被更改时，尝试在IE8中访问win会导致异常。回落操作是重新设置`document.domain`，然后通过document.write注入内联的body onload处理方法并调用它。幸运的是，此代码仅在IE8上需要，因此如果您不需要支持IE8，则可以完全忽略这一块。

调用`document.write()`中一个有趣的hack是将`<body>`标签分割成两个字符串。这样做的话某些表现不佳的第三方解析器（大家都知道说的是谁）便不会将其解析为一个未闭合的标签。 （是的，这是我们遇到的实际问题！）

最后，加载器通过调用`document.close()`，这样就能在代码控制权返回到自己的事件循环里之后触发附加的处理方法。

### 遵循内容安全策略(CSP)

内容安全策略（Content Security Policy）是一种技术，用于指定网站可接受资源（如script脚本）的允许列表。主要用于应对XSS攻击，并且因为[最新”Magecart”信用卡伪造攻击](https://www.govinfosecurity.com/magecart-cybercrime-groups-mass-harvest-payment-card-data-a-11700)的缘故引起了一些关注。 Ryan Barnett最近[深入探讨](https://blogs.akamai.com/sitr/2018/11/protecting-your-website-visitors-from-magecart-trust-but-verify.html)了网站如何使用CSP来保护自己免受此类攻击。

CSP的合规在内联script脚本中表现最佳，如果script内容不发生变化，你可以为它添加一个哈希值。但是当你的script脚本创建一个iframe，其中包含更多脚本时，情况就会变得有点复杂。

对于需要使用iframe回落的浏览器，[Firefox和IE 10和11也支持CSP](https://caniuse.com/#feat=contentsecuritypolicy)，这意味着我们需要一个既支持在当前视口，又支持在iframe中使用preload加载脚本的CSP解决方案。

先前版本的加载器总是使用javascript设置URI作为iframe的src。这是为了解决IE6中的一个bug，`about: blank`会在安全的上下文中触发不安全内容警告。 CSP认为javascript方案与内联事件处理方法类似-为了使用CSP策略，策略中将要求[不安全内联](https://www.w3.org/TR/CSP2/#directive-script-src)（或CSP3中的[不安全哈希](https://www.w3.org/TR/CSP3/#unsafe-hashes-usage)）。如果忽略IE6的支持，我们可以使用`about: blank`URL，这种做法与我们的目的有等同的属性，但不需要在`frame-src`或`script-src`的CSP策略中设置白名单。

同时先前版本的加载器通过`document.write`来绑定事件处理程序。大多数情况下不需要这样做，但这确实为CSP带来了问题。这些匿名的iframe会[继承父窗口的CSP策略](https://www.w3.org/TR/CSP3/#security-inherit-csp)，因此要在写入子窗口内写脚本的话，CSP策略也需要允许该脚本的执行。值得庆幸的是，事实证明这种技术仅与IE8上`document.domain`边界案例相关（不支持CSP），因此对于支持CSP的现代浏览器，我们都可以使用`addEventListener`方法。

最后，使用CSP，还有可能通过`style-src`限制内联样式，甚至是限制通过JavaScript设置内联样式（尽管正在[讨论中](https://github.com/w3c/webappsec-csp/issues/212)）。让加载器通过CSSOM应用iframe样式可以减轻style-src指令引起的潜在问题。
这些变化累积的效果是，只需为`script-src`指令添加两个值-加载器代码的哈希值和script脚本的地址，就可以让加载器的代码段变的更安全。

您可以使用[report-uri.com工具](https://report-uri.com/home/hash)来生成script脚本的哈希值。

### 完整代码

完整代码在[github gist](https://gist.github.com/bluesmoon/675c37368ddd58dfa5f578f1e5a59778)中。它还包含一个iframe子窗口脚本用的dom遍历器，如果script脚本加载并初始化完成之后父级iframe修改了`document.domain`，这时就会用到它。

### 加载器

对于加载器，我们的CSP头看起来像这样：
```
Content-Security-Policy: script-src sha256-N17tpZTa695DVQJ0H+pRpxvMH/27hbTyxTdTugwOGvQ=
```
内联脚本的script节点看起来是这样:
```
<script id="nb-loader-script">
(function(url) {

  // document.currentScript在大多数浏览器中是有效的但也有例外
  var where = document.currentScript || document.getElementById("nb-loader-script"),
      promoted = false,
      LOADER_TIMEOUT = 3000,
      IDPREFIX = "__nb-script";

  // 此函数将一个预加载link元素优化成一个异步的script节点
  function promote() {
    var s;
    s = document.createElement("script");
    s.id = IDPREFIX + "-async";
    
    s.src = url;

    where.parentNode.appendChild(s);
    promoted = true;
  }

  // 此函数用于在不支持预加载提示的浏览器中通过ifram加载script脚本
  function iframe_loader() {
    promoted = true;
    var win, doc, dom, s, bootstrap, iframe = document.createElement("iframe");

    // IE6不支持CSP，将about:blank处理为不安全内容，所以我们需要使用javascript:void(0), 
    // 而在不支持CSP的浏览器中, javascript:void(0)被认为是不安全的内联脚本，所以我们倾向使用about:blank
    iframe.src = "about:blank";
    
    // 通过设置合适的title和role来更好地配合screen reader或其它无障碍辅助技术
    iframe.title = "";
    iframe.role = "presentation";
    
    s = (iframe.frameElement || iframe).style;
    s.width = 0; s.height = 0; s.border = 0; s.display = "none";
    
    where.parentNode.insertBefore(iframe, where);
    try {
      win = iframe.contentWindow;
      doc = win.document.open();
    }
    catch (e) {
      // 这是因为document.domain被修改了，并且当前环境为旧版本IE，所以我们会捕获到一个访问禁止(access denied)错误
      // 注释：唯一会产生这个问题的浏览器不支持CSP
      
      // 我们从父窗口取`document.domain`
      dom = document.domain;
      
      // 将iframe的src属性设为一个javascript脚本的url地址会立刻将它的document.domain设为和父级窗口一样
      // 这个特性让我们能够长时间访问iframe的document以注入我们的script脚本
      // 我们的script脚本后续 可能会需要做更多域名相关的适配
      iframe.src = "javascript:var d=document.open();d.domain='" + dom + "';void(0);";
      win = iframe.contentWindow;
      doc = win.document.open();
    }

    bootstrap = function() {
      // 这段代码在子iframe中运行
      var js = doc.createElement("script");
      js.id = IDPREFIX + "-iframe-async";
      js.src = url;
      doc.body.appendChild(js);
    };
    
    try {
      win._l = bootstrap

      if (win.addEventListener) {
        win.addEventListener("load", win._l, false);
      }
      else if (win.attachEvent) {
        win.attachEvent("onload", win._l);
      }
    }
    catch (f) {
      // 兼容IE8的不安全版本
      // 如果document.domain被修改，则不能用win，但我们可以用doc
      doc._l = function() {
        if (dom) {
          this.domain = dom;
        }
        bootstrap();
      }
      doc.write('<bo' + 'dy onload="document._l();">');
    }
    doc.close();
  }

  // 我们先检查一下当前浏览器是否支持通过link元素实现预加载提示
  var l = document.createElement("link");

  if (l.relList && typeof l.relList.supports === "function" && l.relList.supports("preload") && ("as" in l)) {
    l.href = url;
    l.rel  = "preload";
    l.as   = "script";
    
    // 如果link成功地预加载了我们的script代码，我们将把它修改成一个script节点
    l.addEventListener("load", promote);
    
    // 如果预加载失败或超时，我们将回落到iframe加载的方案
    l.addEventListener("error", iframe_loader);
    setTimeout(function() {
        if (!promoted) {
            iframe_loader();
        }
    }, LOADER_TIMEOUT);
    
    where.parentNode.appendChild(l);
  }
  else {
    // 如果不支持预加载提示，直接回落到iframe加载方案
    iframe_loader();
  }

})("https://your.script.url/goes/here.js");
</script>
```

### DOM 遍历器

把这段代码放在你的脚本开头，任何时候你都能通过调用它安全地访问父窗口
```
(function _check_doc_domain(domain) {
  // 如果iframe创建完成，脚本加载完毕后父窗口修改了document.domain, 这段工具代码用来遍历document.domain

  /*eslint no-unused-vars:0*/
  var test;

  if (!window) {
    return;
  }

  // 如果没有传入domain，将会再全局调用
  // 只有在我们调用自身的时候domain才会传入
  // 因此这里我们跳过了frame的检查
  if (typeof domain === "undefined") {
    // 如果在主窗口执行，我们不需要这样做
    if (window.parent === window || !document.getElementById("__nb-script-iframe-async")) {
      return;// true;  // 什么都不做
    }

    try {
      // 如果document.domain在页面加载的过程中被修改(从www.blah.com改成了blah.com),
      // IE的window.parent.window.location.href会抛出”Permission Denied”的异常.
      // 将里面的domain重设以匹配外面的domain，让地址重新可访问
      if (window.document.domain !== window.parent.window.document.domain) {
        window.document.domain = window.parent.window.document.domain;
      }
    }
    catch (err) {
      // 这个异常我们可以日志记录下来，没什么别的处理可以做
    }
  }

  domain = document.domain;

  if (domain.indexOf(".") === -1) {
    // 到达了顶级域名
    return;// false;  // 依然不行，但我们依旧把能做的都做了
  }

  // 1. 尝试不设置document.domain
  try {
    test = window.parent.document;
    return;// test !== undefined;  // test如果有值就说明正常
  }
  // 2. 设置document.domain
  catch (err) {
    document.domain = domain;
  }
  try {
    test = window.parent.document;
    return;// test !== undefined;  // test如果有值就说明正常
  }
  // 3. 去掉开头部分再次尝试
  catch (err) {
    domain = domain.replace(/^[\w\-]+\./, "");
  }

  _check_doc_domain(domain);
})();
```