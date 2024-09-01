---
title: "Babel transform runtime explained"
date: 2024-08-25
draft: false
categories:
- Coding
tags:
- javascript
- babel
- polyfill
- es6
- plugin-transform-runtime
---

`@babel/plugin-transform-runtime` main goal is to address some issues that arise from Polyfill, Helper, and Regenerator compilation. The following sections will introduce and explain the solutions to these problems one by one.

## Polyfill
People who work on web front-end development should be familiar with this term. If you look up this term in a dictionary, its explanation is some kind of filling material. Before the term Polyfill came into use, there was another term called Shim. What’s the difference between the two? For example, if there’s a hole in a wall and you use a board to cover it, that’s a Shim. If you use plaster to fill and smooth out the hole, that’s a Polyfill. The difference is that with the former, you can clearly see there’s a foreign object, while with the latter, you won’t feel any difference. In terms of code, if you need to solve the issue of different browsers handling the fullscreen API differently: 

Shim([Source](https://www.npmjs.com/package/easy-fullscreen))：

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

Polyfill:
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

Both can achieve fullscreen functionality, but the latter uses the standard API, while the former does not.

#### Global Polyfill 
This is actually the default mode of Babel:

```js
//import '@babel/polyfill'; //deprecated
import 'core-js';

var sym = Symbol();
var promise = Promise.resolve();
```

The principle is quite simple: manually import all Polyfills at the very beginning of the file. Since it is global, essentially, it only needs to be loaded in one file. The common practice is to import it in the entry file. The problem with the method shown in the example above is that it loads a bunch of Polyfills that are not actually used by the project or files. Therefore, core-js provides a more advanced approach:

```js
import 'core-js/es/symbol';
import 'core-js/es/promise';

var sym = Symbol();
var promise = Promise.resolve();
```

Here's the translation:

---

Managing these imports can be cumbersome. For example, if you no longer need `Promise`, it would be more space-efficient to remove it. However, when updating the code, it's easy to forget to remove or import certain things, leading to issues where everything works locally but breaks on older browsers in production. To address this, `@babel/preset-env` offers a feature with a `useBuiltIns` option. If you set it to `entry`, Babel will load the necessary Polyfills based on the browsers you want to support. It will transform:

```js
import "core-js";
```

into (depending on the browser environment settings):

```js
import "core-js/modules/es.string.pad-start";
import "core-js/modules/es.string.pad-end";
```

If you set it to `usage`, it will only load the Polyfills used in the file. For example:

```js
var a = new Promise();
```

will be transformed into:

```js
import 'core-js/es/promise';

var promise = Promise.resolve();
```

But if the target browser environment already supports `Promise`, no Polyfill will be loaded, so it effectively does nothing:

```js
var a = new Promise();
```
The advantage of global Polyfills is that, with the support of bundlers, a project will generally have only one instance of each Polyfill. The downside is that if used in packages, it pollutes the global namespace. If the project itself also uses Polyfills, it can lead to duplicate loading. Imagine a project using this approach and simultaneously using N packages that also follow this approach. In the worst-case scenario, the Polyfills could be loaded N+1 times, with the last one loaded overwriting the previous ones. If the implementations of the Polyfills differ (since it's global and doesn’t necessarily have to use core-js), this can lead to unpredictable issues.

Reference: 
1. [https://babeljs.io/docs/en/babel-preset-env#usebuiltins](https://babeljs.io/docs/en/babel-preset-env#usebuiltins)
2. [https://babeljs.io/docs/en/babel-polyfill](https://babeljs.io/docs/en/babel-polyfill)

#### Scoped ES6 Polyfill  

Due to the issues with the global mode mentioned earlier, Babel provides an alternative mode that requires using `@babel/plugin-transform-runtime` together with `@babel/runtime`:

Babel config:
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
The principle behind it is to use the module features of ES6 to make Polyfills local variables:

```js
var sym = Symbol();

var promise = Promise.resolve();

var check = arr.includes("yeah!");

console.log(arr[Symbol.iterator]());
```
transforms to:
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
The advantage of this approach is that it doesn't pollute the global namespace, but it doesn't fully address the issue of code duplication, unless all the packages used in the project are using the same version of `runtime-corejs`.

Reference: 
1. [https://babeljs.io/docs/en/babel-runtime](https://babeljs.io/docs/en/babel-runtime)
2. [https://babeljs.io/docs/en/babel-plugin-transform-runtime](https://babeljs.io/docs/en/babel-plugin-transform-runtime)

## Helper

Helper functions are pieces of code that Babel generates to support certain ES6 syntax features. For example, for an `async function`:

```js
async function abc() {}
```

Babel will transform it into:

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

In this code, `_asyncToGenerator` is a helper function. It is responsible for converting the `abc` function to have async-like behavior. However, the issue with this approach is that the `_asyncToGenerator` function will be duplicated in every module that uses it.

To address this, `babel-plugin-transform-runtime` provides support. After adding it (following the configuration for Polyfills mentioned earlier), `transform-runtime` will convert the code to use imports, which effectively reduces code duplication:

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

Reference: [https://babeljs.io/docs/en/babel-plugin-transform-runtime](https://babeljs.io/docs/en/babel-plugin-transform-runtime)

## Regenerator

Please checkout MDN for details about Regenerator. `regeneratorRuntime` addresses the issue of polluting the global namespace:

```js
function* foo() {}
```

will produce:

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

However, since `regeneratorRuntime` is a global variable, it can cause pollution. After using `transform-runtime`, it will change to a method that does not pollute the global namespace:

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

Reference: [https://babeljs.io/docs/en/babel-plugin-transform-runtime](https://babeljs.io/docs/en/babel-plugin-transform-runtime)

## Conclution

At this point, you might wonder which approach is better. Essentially, each method has its own use case, which is why Babel offers so many options. After all, if it weren't necessary, everyone would prefer the approach with the least complexity. Regarding usage scenarios, the official Rollup Babel plugin documentation provides an explanation of the settings for `babelHelpers`:

>  * 'runtime' - you should use this especially when building libraries with Rollup. It has to be used in combination with @babel/plugin-transform-runtime and you should also specify @babel/runtime as dependency of your package. Don't forget to tell Rollup to treat the helpers imported from within the @babel/runtime module as external dependencies when bundling for cjs & es formats. This can be accomplished via regex (external: [/@babel\/runtime/]) or a function (external: id => id.includes('@babel/runtime')). It's important to not only specify external: ['@babel/runtime'] since the helpers are imported from nested paths (e.g @babel/runtime/helpers/get) and Rollup will only exclude modules that match strings exactly.
> * 'bundled' - you should use this if you want your resulting bundle to contain those helpers (at most one copy of each). Useful especially if you bundle an application code.
> * 'external' - use this only if you know what you are doing. It will reference helpers on global babelHelpers object. Used in combination with @babel/plugin-external-helpers.
> * 'inline' - this is not recommended. Helpers will be inserted in each file using this option. This can cause serious code duplication. This is the default Babel behavior as Babel operates on isolated files - however, as Rollup is a bundler and is project-aware (and therefore likely operating across multiple input files), the default of this plugin is "bundled".

In short, if you are developing a library, it is generally recommended to use the `transform-runtime` approach. If you are developing an application, it is recommended to use the `bundled` approach provided by Rollup, which essentially combines all `helpers` into a single version. My own guess is that, theoretically, it will add a scope at the application level to ensure that it only applies to and affects the application layer.

