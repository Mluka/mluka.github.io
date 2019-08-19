---
layout:     post
title:      "【翻译】globalThis 全知道"
subtitle:   "补丁上面打补丁，globalThis 实属无奈之举"
date:       2019-08-19
author:     "MeloGuo"
header-img: "img/post-bg-js-version.jpg"
tags:
    - 翻译
    - JavaScript
---

由 Jordan Harband 提出的 ECMAScript 的新特性 `globalThis` 提供了一种获取全局对象的标准方式。

## JavaScript 的全局作用域
JavaScript 的作用域从全局作用域开始为节点，形成了一个嵌套的树形结构。 仅仅在浏览器里直接使用 `<script>` 标签时的代码直接运行在全局作用域中。在这当中有两种全局变量：
* 普通的*全局声明变量*
	* 它们通过 `const`, `let` 和 `class` 声明被创建在脚本的最顶层作用域
* 存放在*全局对象*中的*全局对象变量*
	* 它们通过 `var` 和函数声明被创建在脚本的最顶层作用域
	* 它们能够通过全局对象被创建、被删除以及被读取
	* 除了以上两点，它们就和普通变量没什么区别

全局对象能够通过 `globalThis` 访问到。下面的 HTML 代码示范了 `globalThis` 和几种不同的全局变量。

```html
<script>
  const one = 1;
  var two = 2;
</script>  
<script>
  // 所有的 script 标签中都共享同一个顶层作用域
  console.log(one); // 1
  console.log(two); // 2

  // 不是所有的声明都在全局对象上创建了属性
  console.log(globalThis.one); // undefined
  console.log(globalThis.two); // 2
</script>
```

应注意的是，每一个模块都有自己的作用域。因此，在模块中的顶层作用域的变量不是全局的。下图展示了不同作用域之间的关系。

![](https://ws1.sinaimg.cn/large/0070gOERly1g64nq6lp38j30kt0btwfd.jpg)

## this 的值
无论何时调用方法的对象中， `this` 的值都取决于当前的作用域：
* 顶级作用域中：全局对象
* ES 模块的顶层中：undefined
* 在函数调用中：
	* 严格模式（包括模块）：undefined
	* 松散模式：同全局 `this`

如果你间接的调用 `eval()`，它将会随意的在全局作用域中执行。因此，你能用下面的代码来获取全局的 `this`：
```javascript
const theGlobalThis = eval.call(undefined, this);
```
需要警告的是，如果你使用了 CSP (Content Security Policy)， `eval`, `new Function()` 等方法是无法生效的。这就使得这种获取全局对象的方法在一些情况下是不合适的。

## globalThis 和其他（window 等等）
`globalThis` 是一个全新的标准方法用来获取全局 `this` 。之前开发者会通过如下的一些方法获取：
* 全局变量 `window`：是一个经典的获取全局对象的方法。但是它在 Node.js 和 Web Workers 中并不能使用
* 全局变量 `self`：通常只在 Web Workers 和浏览器中生效。但是它不支持 Node.js。一些人会通过判断 `self` 是否存在识别代码是否运行在 Web Workers 和浏览器中
* 全局变量 `global`：只在 Node.js 中生效

新提案也规定了，`Object.prototype` 必须在全局对象的原型链中。下面的代码在现在的最新浏览器中已经会返回 `true` 了：
```javascript
Object.prototype.isPrototypeOf(globalThis); // true
```

## globalThis 使用实例
由于需要向下兼容，全局对象现在已经被认为是一个 JavaScript 无法摆脱的错误了。它会造成负面的性能影响并且令人产生困惑。

ECMAScript 介绍了一些方法来让你更简单的避免触碰全局对象 - 例如：
* `const` 、`let` 和 `class` 声明被使用在全局作用域时也不会创建全局对象属性
* 每个 ECMAScript 模块他都它自身的作用域
将全局变量只作为一个变量来处理要比作为 `globalThis` 的属性来处理要合适的多。这个特性已经在所有的 JavaScript 平台上都能工作了。

因此，这有几个使用 `globalThis` 相关的场景 - 例如：
* 在 polyfills 和 shims 中提供 JavaScript 引擎新特性
* 通过特性检测，去找出一个 JavaScript 引擎支持哪种特性

## polyfill
此提案的作者 Jordan Harband 写过一个 `globalThis` 的 polyfill。
用 CommonJS 语法引入：
```javascript
// 生成 `global` 变量的值
var global = require('globalthis')();

// 垫片 `global` （全局安装）
require('globalthis/shim')();
```
用 ES6 模块语法引入：
```javascript
// 生成 `global` 变量的值
import getGlobal from 'globalthis';
const global = getGlobal();

// 垫片 `global` （全局安装）
import shim from 'globalthis/shim'; shim();
```
这个包总是会使用“最原生”的方法来访问全局对象（在 Node.js 中是 `global`，在浏览器中是 `window` 等等）

## 得到一个全局对象的引用
在内部实现中 polyfill 使用 `getPolyfill()` 来得到全局对象。下面是实现方法：
```javascript
// polyfill.js
var implementation = require('./implementation');

module.exports = function getPolyfill() {
    if (typeof global !== 'object' || !global
        || global.Math !== Math || global.Array !== Array) {
        return implementation;
    }
    return global;
};
```

```javascript
// implementation.js
if (typeof self !== 'undefined') {
  module.exports = self;
} else if (typeof window !== 'undefined') {
  module.exports = window;
} else if (typeof global !== 'undefined') {
  module.exports = global;
} else {
  module.exports = Function('return this')();
}
```

## FAQ: globalThis
### 为什么不在所有地方都使用 global、self 或 window？
唉，这是不可能的，因为许多 JavaScript 库使用这些变量来检测代码运行在哪一个平台上。

### globalThis 有考虑使用过其他名字吗？
这个 [issue](https://github.com/tc39/proposal-global/issues/32)列出了我们考虑过的所有命名以及没有采用的原因。

## 资源与背景资料
**General background**:
* A general overview of global variables: [section “Global variables”](https://exploringjs.com/impatient-js/ch_variables-assignment.html#global-variables) in “JavaScript for impatient programmers”
* In-depth information on how JavaScript manages its global variables: [“How do JavaScript’s global variables really work?”](https://2ality.com/2019/07/global-scope.html) on 2ality
* Useful background for what happens in browsers: [“Defining the WindowProxy, Window, and Location objects”](https://blog.whatwg.org/windowproxy-window-and-location) by Anne van Kesteren
* Various ways of accessing the globalthisvalue: [“A horrifyingglobalThispolyfill in universal JavaScript”](https://mathiasbynens.be/notes/globalthis) by Mathias Bynens

**Specifications**:
* Very technical: [“Realms, settings objects, and global objects”](https://html.spec.whatwg.org/multipage/webappapis.html#realms-settings-objects-global-objects) in the WHATWG HTML standard
* In the ECMAScript specification, you can see how web browsers customize globalthis: [“InitializeHostDefinedRealm()”](https://tc39.es/ecma262/#sec-initializehostdefinedrealm) 

#### 著作权声明

本文译自 [ES proposal: globalThis](https://2ality.com/2019/08/global-this.html)   
译者 [郭梓梁](https://www.zhihu.com/people/mluka/activities)，首次发布于 [MeloGuo Blog](http://meloguo.com)，转载请保留以上链接
