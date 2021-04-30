---
title: "詳解 @babel/plugin-transform-runtime"
date: 2021-04-29
draft: false
categories:
- 寫扣
tags:
- javascript
- babel
- polyfill
- es6
- plugin-transform-runtime
---

`@babel/plugin-transform-runtime` 主要是要解決 Polyfill、Helper 和 Regenerator 編譯產生的一些問題，下面會逐一介紹和講解其解決的方法。

## Polyfill
有在寫網頁前端的人對這名詞應該都不陌生，如果你丟這個詞去查字典的話，它的解釋是某種填充材料，在 Polyfill 這個名詞出現以前有另一個名詞叫 Shim， 同樣如果去查字典的解釋是墊片，兩者之間有什麼不同呢？舉個例子，假如一面牆上破了一個洞，你用一塊板子釘上去蓋住它那就是 Shim，如果你用補土把它補好補滿那就是 Polyfill，差異在於前者你可以很明顯知道有個不一樣的外來物，後者你不會有感覺。換成程式碼的話，如果要解決不同瀏覽器處理 fullscreen api 差異的問題:

Shim 形式([Source](https://www.npmjs.com/package/easy-fullscreen))：

```js
import Fullscreen from 'easy-fullscreen';

if (Fullscreen.isEnabled) {
  fullscreenButton.onclick = function () {
    // check if is in fullscreen mode
    if (Fullscreen.isFullscreen) {
      // exit fullscreen
      Fullscreen.exit();
    } else {
      // request to enter fullscreen
      Fullscreen.request(fullscreenElement);
    }
  };
}
```

Polyfill 形式：
```js
import 'core-js';

if (document.documentElement.requestFullscreen) {
  fullscreenButton.onclick = function () {
    if (document.fullscreenElement) {
      document.exitFullscreen();
    } else {
      document.documentElement.requestFullscreen();
    }
  };
}
```

兩者都可以達成 fullscreen 的功能，但後者用的是標準 API，前者則不是。

#### Polyfill 傳統全域
這個其實就是 Babel 預設的模式:
```js
//import '@babel/polyfill'; //淘汰中
import 'core-js';

var sym = Symbol();
var promise = Promise.resolve();
```

原理相當簡單，就是在檔案的一開頭手動把所有 Polyfill 全部載進來，因爲是全局的，基本上只要其中一個檔案有載入就好了，慣例做法就是在 entry 檔案裡 import。上述範例的做法問題就是會載入一堆專案/檔案根本沒用到的 Polyfill，於是 core-js 就提供了較進階的用法

```js
import 'core-js/es/symbol';
import 'core-js/es/promise';

var sym = Symbol();
var promise = Promise.resolve();
```

但管理這些載入很煩，例如之後沒有要用到 Promise 了，其實拿掉會比較省空間，但改程式時常常就會忘記，甚至忘了 import，造成在本機測沒問題，上線後在較舊的瀏覽器爆炸的問題，於是就有了搭配 `@babel/preset-env` 的方式，其中有個 `useBuiltIns` 的設定，設成 `entry` 的話，Babel 就會根據你設定想要支援的瀏覽器載入該瀏覽器環境缺失的 Polyfill，把
```js
import "core-js";
```
轉成(取決於瀏覽器環境設定)
```js
import "core-js/modules/es.string.pad-start";
import "core-js/modules/es.string.pad-end";
```
設成 `usage` 的話就只會載入該檔案會用到的
```js
var a = new Promise();
```
轉成
```js
import 'core-js/es/promise';

var promise = Promise.resolve();
```
但如果設定的瀏覽器環境本來就支援 Promise 的話，則不會載入，所以什麼都不會做
```js
var a = new Promise();
```

傳統全局的好處是在打包工具的支援下，基本上一個專案所有的 Polyfill 都只會有一份，壞處則是如果是用在套件的話會污染全局，假設用套件的專案本身也有在用 Polyfill 的話，就會造成重複載入，想像一下一個專案本身用了這種模式，同時用的 N 個套件也都是這種模式，那麼 Polyfill 最糟的情況下就會被載入 N+1 次，而且最後一個被載入會覆蓋掉前面的，假設 Polyfill 實作方式又有所不同的話(因爲是全局，不一定要用 core-js)就會造成難以預期的問題。

參考：
1. [https://babeljs.io/docs/en/babel-preset-env#usebuiltins](https://babeljs.io/docs/en/babel-preset-env#usebuiltins)
2. [https://babeljs.io/docs/en/babel-polyfill](https://babeljs.io/docs/en/babel-polyfill)

#### Polyfill ES6 局域
因爲前面全域模式的問題，所以 Babel 提供了另一種模式，需要`@babel/plugin-transform-runtime`搭配`@babel/runtime`來一起用

Babel 設定：
```js
{
  "plugins": [
    [
      "@babel/plugin-transform-runtime",
      {
        "corejs": true
      }
    ]
  ]
}
```

其運作原理就是利用 ES6 的模組特性把 Polyfill 變成局域的變數

```js
var sym = Symbol();

var promise = Promise.resolve();

var check = arr.includes("yeah!");

console.log(arr[Symbol.iterator]());
```
會轉成
```js
import _getIterator from "@babel/runtime-corejs3/core-js/get-iterator";
import _includesInstanceProperty from "@babel/runtime-corejs3/core-js-stable/instance/includes";
import _Promise from "@babel/runtime-corejs3/core-js-stable/promise";
import _Symbol from "@babel/runtime-corejs3/core-js-stable/symbol";

var sym = _Symbol();

var promise = _Promise.resolve();

var check = _includesInstanceProperty(arr).call(arr, "yeah!");

console.log(_getIterator(arr));
```

這樣做的好處是不會污染全域，但比較無法解決程式碼重複的問題，除非專案用的所有套件都用同一個 runtime-corejs 版本。

參考：
1. [https://babeljs.io/docs/en/babel-runtime](https://babeljs.io/docs/en/babel-runtime)
2. [https://babeljs.io/docs/en/babel-plugin-transform-runtime](https://babeljs.io/docs/en/babel-plugin-transform-runtime)

## Helper

Helper 基本上就是 Babel 爲了要達成 ES6 的一些語法支援而產生的一些程式，例如 async function

```js
async function abc() {}
```
會轉成
```js
"use strict";

function asyncGeneratorStep(gen, resolve, reject, _next, _throw, key, arg) { try { var info = gen[key](arg); var value = info.value; } catch (error) { reject(error); return; } if (info.done) { resolve(value); } else { Promise.resolve(value).then(_next, _throw); } }

function _asyncToGenerator(fn) { return function () { var self = this, args = arguments; return new Promise(function (resolve, reject) { var gen = fn.apply(self, args); function _next(value) { asyncGeneratorStep(gen, resolve, reject, _next, _throw, "next", value); } function _throw(err) { asyncGeneratorStep(gen, resolve, reject, _next, _throw, "throw", err); } _next(undefined); }); }; }

function abc() {
  return _abc.apply(this, arguments);
}

function _abc() {
  _abc = _asyncToGenerator( /*#__PURE__*/regeneratorRuntime.mark(function _callee() {
    return regeneratorRuntime.wrap(function _callee$(_context) {
      while (1) {
        switch (_context.prev = _context.next) {
          case 0:
          case "end":
            return _context.stop();
        }
      }
    }, _callee);
  }));
  return _abc.apply(this, arguments);
}
```

其中 `_asyncToGenerator` 就是 helper，它負責把 abc 這個 function 改成有像 async 一樣的特性/行爲，但這樣的問題是 `_asyncToGenerator` 這個函數在每個用到它的模組都會有一份重複的。這時候 `babel-plugin-transform-runtime` 又要來支援了，在加入後(參考上面Polyfill部分設定)，`transform-runtime` 會把上述程式碼轉換成引入的方式就可以有效減少重複的程式碼

```js
"use strict";

var _interopRequireDefault = require("@babel/runtime/helpers/interopRequireDefault");

var _regenerator = _interopRequireDefault(require("@babel/runtime/regenerator"));

var _asyncToGenerator2 = _interopRequireDefault(require("@babel/runtime/helpers/asyncToGenerator"));

function abc() {
  return _abc.apply(this, arguments);
}

function _abc() {
  _abc = (0, _asyncToGenerator2.default)( /*#__PURE__*/_regenerator.default.mark(function _callee() {
    return _regenerator.default.wrap(function _callee$(_context) {
      while (1) {
        switch (_context.prev = _context.next) {
          case 0:
          case "end":
            return _context.stop();
        }
      }
    }, _callee);
  }));
  return _abc.apply(this, arguments);
}
```

參考：[https://babeljs.io/docs/en/babel-plugin-transform-runtime](https://babeljs.io/docs/en/babel-plugin-transform-runtime)

## Regenerator

這個不想介紹了，請直接參考 MDN。Regenerator 要處理的問題是污染全域的問題

```js
function* foo() {}
```

會產生
```js
"use strict";

var _marked = [foo].map(regeneratorRuntime.mark);

function foo() {
  return regeneratorRuntime.wrap(
    function foo$(_context) {
      while (1) {
        switch ((_context.prev = _context.next)) {
          case 0:
          case "end":
            return _context.stop();
        }
      }
    },
    _marked[0],
    this
  );
}
```

但因爲 `regeneratorRuntime` 是全域變數，所以會被污染，用了 `tranfrom-runtime` 後會變成不會污染全域變數的方式

```js
"use strict";

var _regenerator = require("@babel/runtime/regenerator");

var _regenerator2 = _interopRequireDefault(_regenerator);

function _interopRequireDefault(obj) {
  return obj && obj.__esModule ? obj : { default: obj };
}

var _marked = [foo].map(_regenerator2.default.mark);

function foo() {
  return _regenerator2.default.wrap(
    function foo$(_context) {
      while (1) {
        switch ((_context.prev = _context.next)) {
          case 0:
          case "end":
            return _context.stop();
        }
      }
    },
    _marked[0],
    this
  );
}
```

參考： [https://babeljs.io/docs/en/babel-plugin-transform-runtime](https://babeljs.io/docs/en/babel-plugin-transform-runtime)

## 總結

看到這裡可能有個疑問就是，到底那種方式比較好，基本上所有的方式都有其應用的場景，不然 babel 也不會生出那麼多種選項，畢竟沒有必要的話，大家都會比較傾向複雜度越低越好。關於使用情景，Rollp 官方的 babel-plugin 對 babalHelpers 的設定就有說明：


>  * 'runtime' - you should use this especially when building libraries with Rollup. It has to be used in combination with @babel/plugin-transform-runtime and you should also specify @babel/runtime as dependency of your package. Don't forget to tell Rollup to treat the helpers imported from within the @babel/runtime module as external dependencies when bundling for cjs & es formats. This can be accomplished via regex (external: [/@babel\/runtime/]) or a function (external: id => id.includes('@babel/runtime')). It's important to not only specify external: ['@babel/runtime'] since the helpers are imported from nested paths (e.g @babel/runtime/helpers/get) and Rollup will only exclude modules that match strings exactly.
> * 'bundled' - you should use this if you want your resulting bundle to contain those helpers (at most one copy of each). Useful especially if you bundle an application code.
> * 'external' - use this only if you know what you are doing. It will reference helpers on global babelHelpers object. Used in combination with @babel/plugin-external-helpers.
> * 'inline' - this is not recommended. Helpers will be inserted in each file using this option. This can cause serious code duplication. This is the default Babel behavior as Babel operates on isolated files - however, as Rollup is a bundler and is project-aware (and therefore likely operating across multiple input files), the default of this plugin is "bundled".

簡單來說如果是開發套件的話，比較建議用 `transform-runtime` 方式，如果是開發應用的話，建議用 Rollup 自己提供的 `bundled` 方式，基本上就是把所有 helpers 用成一份，我自己猜測理論上會在應用層加一個 scope 來保證只會套用/影響到應用層。以上就簡單說明，只是現實世界的具體情況會更複雜，之後會再寫一篇開發/載入模組時要注意跟可以優化的事項。

