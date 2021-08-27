> 最近几年微前端的兴起，带火了[single-spa](https://github.com/single-spa/single-spa)，很多微前端的技术方案都是基于它去实现的。但是single-spa自身并没有实现模块加载的能力，官方在[The Recommended Setup](https://single-spa.js.org/docs/recommended-setup/#systemjs)中推荐了使用SystemJS来进行微前端应用的加载。

在SystemJS的[GitHub](https://github.com/systemjs/systemjs)主页上，关于自己是这么介绍的：

> Dynamic ES module loader.

也就是说，它对自己的定位是一个动态的ES Module加载器。SystemJS主要实现了：

1. 兼容ES module的模块格式，能够在ES5的环境中执行
2. 基于import-maps的模块映射

所以，在开始分析SystemJS之前，我们先来介绍一下ES module和Import-Maps.

## ES Module

###### 在ES6，也就是ES2015之后，ES Module已经逐渐成为了前端开发的模块标准。关于ES Module的更多知识，可以参考阮一峰的教程——[Module的语法](https://es6.ruanyifeng.com/#docs/module).

### 模块加载

在较新的浏览器中，我们可以在script标签中，可以通过指定`type="module"`来直接加载ES module格式的JavaScript代码文件，例如：

```javascript
// hello.js
export default function sayHello() {
  const result = 'Hello, ES module!'
  console.log(result)
  return result
}

function welcome(user) {
  const result = 'Welcome to es module, ' + user
  console.log(result)
  return result
}

export {
  welcome
}
```

```html
  <script type="module">
    // 直接引入hello.js
    import sayHello, { welcome } from './hello.js'
    // 动态引入hello.js
    import('./hello.js').then(module => {
      const { default: sayHello, welcome } = module
    })
  </script>
```

最近在构建工具领域内很火的[vite](https://github.com/vitejs/vite)和[snowpack](https://github.com/snowpackjs/snowpack), 也是利用了这种特性去实现的。

### import.meta

`import.meta`对象属于ECMAScript标准的一部分，是一个给JavaScript模块暴露特定上下文的元数据属性的对象，它带有一个[`null`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/null)的原型对象。

import.meta包含了这个模块的信息，比如说url. 同时，这个对象是可以扩展的，并且它的属性都是可写，可配置和可枚举的。

获取url:

```javascript
<script type="module" src="my-module.mjs"></script>
console.log(import.meta.url); // { url: "file:///home/user/my-module.mjs" }
```

在Node.js的文档中，还提到了一个[实验性的特性](https://nodejs.org/api/esm.html#esm_import_meta_resolve_specifier_parent)——`import.meta.resolve`:

```javascript
const dependencyAsset = await import.meta.resolve('component-lib/asset.css');
await import.meta.resolve('./dep', import.meta.url);
```

### 支持情况

具体的浏览器支持情况，可以参见[JavaScript  modules via script tag](https://caniuse.com/?search=type%3D%22module%22)；关于Node.js的支持，具体可以参见[Modules:ECMA modules](https://nodejs.org/api/esm.html#esm_modules_ecmascript_modules).

受限于浏览器对于ES module的兼容性，即使是vite/snowpack在build的时候，也是基于rollup/webpack去实现的。而SystemJS，则提供了一种在ES5环境下兼容ES module的格式，方便我们来进行模块的加载。

## 关于Import-Map

在介绍SystemJS之前，还有一个重要的概念——import map. 它实际上是WICG——也就是[Web Platform Incubator Community Group](https://www.w3.org/community/wicg/)发布的规范。它可以通过对于module名称和url之间的映射，实现模块的引入：

引入的模块：

```javascript
import moment from "moment";
import { partition } from "lodash";
```

如果我们对模块做了如下的映射配置：

```html
<script type="importmap">
{
  "imports": {
    "moment": "/node_modules/moment/src/moment.js",
    "lodash": "/node_modules/lodash-es/lodash.js"
  }
}
</script>

```

那么，我们的实际的代码其实如下：

```javascript
import moment from "/node_modules/moment/src/moment.js";
import { partition } from "/node_modules/lodash-es/lodash.js";
```

此外，import-map还规定了对于`scope`的支持，例如：

```javascript
{
  "imports": {
    "querystringify": "/node_modules/querystringify/index.js"
  },
  "scopes": {
    "/node_modules/socksjs-client/": {
      "querystringify": "/node_modules/socksjs-client/querystringify/index.js"
    }
  }
}
```

更多的细节，可以参考import-map的[GitHub](https://github.com/WICG/import-maps)和wicg的[import-maps草案](https://wicg.github.io/import-maps/).

额外的一点，就是Deno已经[内置了对于import-maps的支持](https://deno.land/manual/linking_to_external_code/import_maps)：

```typescript
// import_map.json
{
   "imports": {
      "fmt/": "https://deno.land/std@0.106.0/fmt/"
   }
}

// color.ts
import { red } from "fmt/colors.ts";

console.log(red("hello world"));
```





## 关于SystemJS

SystemJS是一个基于标准的、插件化的模块加载器。它可以做到：

- 将原生的ES module的JavaScript代码转换成[System.register模块格式](https://github.com/systemjs/systemjs/blob/master/docs/system-register.md)，以在不能支持ES module的旧浏览器（最低是IE11）中运行我们的代码
- 借助于[ImportMap](https://github.com/WICG/import-maps)，实现了module名称和module的url的映射，以及通过名称来加载模块

### 性能

官方也宣称SystemJS的执行，几乎可以达到原生模块的运行速度，如下是官方在GitHub上列出的[performance信息](https://github.com/systemjs/systemjs#performance)：

| Tool            | Uncached | Cached |
| --------------- | -------- | ------ |
| Native modules  | 1668ms   | 49ms   |
| SystemJS        | 2334ms   | 81ms   |
| es-module-shims | 2671ms   | 602ms  |

### 支持格式

| Module Format                                                | s.js                                               | system.js                        | File Extension |
| ------------------------------------------------------------ | -------------------------------------------------- | -------------------------------- | -------------- |
| [System.register](/docs/system-register.md)                  | :heavy_check_mark:                                 | :heavy_check_mark:               | *              |
| [JSON Modules](https://github.com/whatwg/html/pull/4407)     | [Module Types extra](/dist/extras/module-types.js) | :heavy_check_mark:               | *.json         |
| [CSS Modules](https://github.com/w3c/webcomponents/blob/gh-pages/proposals/css-modules-v1-explainer.md) | [Module Types extra](/dist/extras/module-types.js) | :heavy_check_mark:               | *.css          |
| [Web Assembly](https://github.com/WebAssembly/esm-integration/tree/master/proposals/esm-integration) | [Module Types extra](/dist/extras/module-types.js) | :heavy_check_mark:               | *.wasm         |
| Global variable                                              | [global extra](/dist/extras/global.js)             | :heavy_check_mark:               | *              |
| [AMD](https://github.com/amdjs/amdjs-api/wiki/AMD)           | [AMD extra](/dist/extras/amd.js)                   | [AMD extra](/dist/extras/amd.js) | *              |
| [UMD](https://github.com/umdjs/umd)                          | [AMD extra](/dist/extras/amd.js)                   | [AMD extra](/dist/extras/amd.js) | *              |

此外，通过相关插件，还可以支持：

- coffeeScript，通过[system-coffee](https://github.com/forresto/system-coffee)
- TypeScript，通过[plugin-typescript](https://github.com/frankwallis/plugin-typescript)
- ...

### 主要API

#### 加载模块

我们可以在html中直接通过script标签，设置`type="systemjs-module"`进行引入：

```html
// 引入systemjs
<script src="system.js"></script>
// 用type="systemjs-module"去标识引入的JavaScript代码是System.register格式
<script type="systemjs-module" src="/js/main.js"></script>
<script type="systemjs-module" src="import:name-of-module"></script>
```

如果需要动态引入的话，我们可以使用`System.import`:

```javascript
// 必须是一个System.register格式的模块
System.import('/aSystemRegisterModule.js');
```

#### 注册模块

我们可以通过System.register来注册一个兼容ES module的模块格式，它具有以下特性：

- 顶层await
- 动态引入
- 循环引用
- 实时绑定
- import.meta.url
- 模块类型
- 完整性
- CSP支持

```javascript
System.register([...deps...], function (_export, _context) {
  return {
    setters: [...setters...],
    execute: function () {

    }
  };
});
```

- deps：模块的依赖
- setters：模块依赖的回调，和deps一一对应；当依赖更新时调用
- _export：`(name: String, value: any) => value`，用来导出绑定的函数
- _context：
  - _context.meta 模块执行中的`import.meta`的值
  - _context.import 类似于`import()`



### 对import-maps的支持

如果我们需要在SystemJS中使用import-maps，则需要在指定script标签为`type="systemjs-importmap"`，并作如下配置：

```html
<script type="systemjs-importmap">
  {
    "imports": {
      "moduleA": "https://xxx/moduleA.js"
      "moduleB": "https://xxx/moduleB.js"
    }
  }
</script>
```

这样，我们就可以直接通过如下方式去引入这些模块了：

```javascript
import * from 'moduleA'
import { funcA, funB } from 'moduleB'
```



### 与构建工具的整合

相对于直接使用ESM格式的JavaScript，我们需要直接把我们的应用编译成[System.register format](https://github.com/systemjs/systemjs/blob/master/docs/system-register.md)，这样才能直接被SystemJS识别。

在我们的构建工具中，我们也需要配置对应的输出格式：

- 在webpack中，设置`libraryTarget`为`"system"`
- 在rollup中，设置`format`为`"system"`



## 参考文档：

[systemjs](https://github.com/systemjs/systemjs)

[Module的语法](