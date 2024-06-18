## 分析打包后文件

### 定位大体积文件

1. 配置webpack-bundle-analyzer 分析打包后文件

![](/assets/img/webpack.png)

打包后可以看大那些文件比较大，划到右侧的treemap图上可以看到大文件中具体包含哪些文件，对于相同的js 文件重复引用，第三方包比较大问题进行优化。

可以考虑从减少单个包的大小减少加载时间，减少文件数量合并小文件减少请求数量方面优化。

## 优化打包后文件

### 使用 SplitChunks提升应用性能

Webpack 默认会将尽可能多的模块代码打包在一起，优点是能减少最终页面的 HTTP 请求数，但缺点也很明显：

1. 页面初始代码包过大，影响首屏渲染性能；
2. 无法有效应用浏览器缓存，特别对于 NPM 包这类变动较少的代码，业务代码哪怕改了一行都会导致 NPM 包缓存失效。

为此，Webpack 提供了 SplitChunksPlugin 插件，专门用于根据产物包的体积、引用次数等做分包优化，规避上述问题，特别适合生产环境使用。

Webpack 默认会将以下三种模块做分包处理：

1. Initial Chunk：entry 模块及相应子模块打包成 Initial Chunk；
2. Async Chunk：通过 import('./xx') 等语句导入的异步模块及相应子模块组成的 Async Chunk；
3. Runtime Chunk：运行时代码抽离成 Runtime Chunk，可通过 entry.runtime 配置项实现。

Runtime Chunk 规则比较简单，但 Initial Chunk 与 Async Chunk 这种略显粗暴的规则会带来两个明显问题：

1. 模块重复打包：

    假如多个 Chunk 同时依赖同一个 Module，那么这个 Module 会被不受限制地重复打包进这些 Chunk

2. 资源冗余

    Webpack 会将 Entry 模块、异步模块所有代码都打进同一个单独的包，这在小型项目通常不会有明显的性能问题，但伴随着项目的推进，包体积逐步增长可能会导致应用的响应耗时越来越长。客户端必须等待整个应用的代码包都加载完毕才能启动运行，但可能用户当下访问的内容只需要使用其中一部分代码。客户端必须等待整个应用的代码包都加载完毕才能启动运行，但可能用户当下访问的内容只需要使用其中一部分代码。

    为此，Webpack 专门提供了 SplitChunksPlugin 插件，用于实现更灵活、可配置的分包，提升应用性能。

SplitChunksPlugin 的用法比较抽象，主要能力有：

1. SplitChunksPlugin 支持根据 Module 路径、Module 被引用次数、Chunk 大小、Chunk 请求数等决定是否对 Chunk 做进一步拆解，这些决策都可以通过 optimization.splitChunks 相应配置项调整定制，基于这些能力我们可以实现：
    
    * 单独打包某些特定路径的内容，例如 node_modules 打包为 vendors；

    * 单独打包使用频率较高的文件

2. SplitChunksPlugin 还提供了 optimization.splitChunks.cacheGroup 概念，用于对不同特点的资源做分组处理，并为这些分组设置更有针对性的分包规则；

3. SplitChunksPlugin 还内置了 default 与 defaultVendors 两个 cacheGroup，提供一些开箱即用的分包特性：

    * node_modules 资源会命中 defaultVendors 规则，并被单独打包；
    * 只有包体超过 20kb 的 Chunk 才会被单独打包；
    * 加载 Async Chunk 所需请求数不得超过 30；
    * 加载 Initial Chunk 所需请求数不得超过 30。

修改 optimization.splitChunks 配置项即可实现自定义的分包策略：

```
module.exports = {
  //...
  optimization: {
    splitChunks: {
      // ...
    },
  },
}
```
splitChunks 主要有两种类型的配置：
 * minChunks/minSize/maxInitialRequest 等分包条件，满足这些条件的模块都会被执行分包；

 * cacheGroup ：用于为特定资源声明特定分包条件，例如可以为 node_modules 包设定更宽松的分包条件。

#### 设置分包范围

首先，SplitChunksPlugin 默认情况下只对 Async Chunk 生效，我们可以通过 splitChunks.chunks 调整作用范围，该配置项支持如下值：
 
 * 字符串 'all' ：对 Initial Chunk 与 Async Chunk 都生效，建议优先使用该值；
 * 字符串 'initial' ：只对 Initial Chunk 生效；
 * 字符串 'async' ：只对 Async Chunk 生效；
 * 函数 (chunk) => boolean ：该函数返回 true 时生效；

```
module.exports = {
  //...
  optimization: {
    splitChunks: {
      chunks: 'all',
    },
  },
}
```
设置为 all 效果最佳，此时 Initial Chunk、Async Chunk 都会被 SplitChunksPlugin 插件优化。

#### 根据 Module 使用频率分包

SplitChunksPlugin 支持按 Module 被 Chunk 引用的次数决定是否分包，借助这种能力我们可以轻易将那些被频繁使用的模块打包成独立文件，减少代码重复。

用法很简单，只需用 splitChunks.minChunks 配置项设定最小引用次数

```
module.exports = {
  //...
  optimization: {
    splitChunks: {
      // 设定引用次数超过 2 的模块才进行分包
      minChunks: 2
    },
  },
}
```
#### 限制分包数量

在 minChunks 基础上，为防止最终产物文件数量过多导致 HTTP 网络请求数剧增，反而降低应用性能，Webpack 还提供了 maxInitialRequest/maxAsyncRequest 配置项，用于限制分包数量：

 * maxInitialRequest：用于设置 Initial Chunk 最大并行请求数；
 * maxAsyncRequests：用于设置 Async Chunk 最大并行请求数。

#### 限制分包体积

除上面介绍的 minChunks —— 模块被引用次数，以及 maxXXXRequest —— 包数量，这两个条件外，Webpack 还提供了一系列与 Chunk 大小有关的分包判定规则，借助这些规则我们可以实现当包体过小时直接取消分包 —— 防止产物过"碎"；当包体过大时尝试对 Chunk 再做拆解 —— 避免单个 Chunk 过大。

 * minSize： 超过这个尺寸的 Chunk 才会正式被分包；
maxSize： 超过这个尺寸的 Chunk 会尝试进一步拆分出更小的 Chunk；
 * maxAsyncSize： 与 maxSize 功能类似，但只对异步引入的模块生效；
 * maxInitialSize： 与 maxSize 类似，但只对 entry 配置的入口模块生效；
 * enforceSizeThreshold： 超过这个尺寸的 Chunk 会被强制分包，忽略上述其它 Size 限制。
 
#### 缓存组 cacheGroups 简介

上述 minChunks、maxInitialRequest、minSize 都属于分包条件，决定是否对什么情况下对那些 Module 做分包处理。此外， SplitChunksPlugin 还提供了 cacheGroups 配置项用于为不同文件组设置不同的规则

```
module.exports = {
  //...
  optimization: {
    splitChunks: {
      cacheGroups: {
        vendors: {
            test: /[\\/]node_modules[\\/]/,
            minChunks: 1,
            minSize: 0
        }
      },
    },
  },
};
```
示例通过 cacheGroups 属性设置 vendors 缓存组，所有命中 vendors.test 规则的模块都会被归类 vendors 分组，优先应用该组下的 minChunks、minSize 等分包配置。

cacheGroups 支持上述 minSice/minChunks/maxInitialRequest 等条件配置，此外还支持一些与分组逻辑强相关的属性，包括：

 * test：接受正则表达式、函数及字符串，所有符合 test 判断的 Module 或 Chunk 都会被分到该组；
 * type：接受正则表达式、函数及字符串，与 test 类似均用于筛选分组命中的模块，区别是它判断的依据是文件类型而不是文件名，例如 type = 'json' 会命中所有 JSON 文件；
 * idHint：字符串型，用于设置 Chunk ID，它还会被追加到最终产物文件名中，例如 idHint = 'vendors' 时，输出产物文件名形如 vendors-xxx-xxx.js ；
 * priority：数字型，用于设置该分组的优先级，若模块命中多个缓存组，则优先被分到 priority 更大的组。

缓存组的作用在于能为不同类型的资源设置更具适用性的分包规则，一个典型场景是将所有 node_modules 下的模块统一打包到 vendors 产物，从而实现第三方库与业务代码的分离。

Webpack 提供了两个开箱即用的 cacheGroups，分别命名为 default 与 defaultVendors，默认配置：

```
module.exports = {
  //...
  optimization: {
    splitChunks: {
      cacheGroups: {
        default: {
          idHint: "",
          reuseExistingChunk: true,
          minChunks: 2,
          priority: -20
        },
        defaultVendors: {
          idHint: "vendors",
          reuseExistingChunk: true,
          test: /[\\/]node_modules[\\/]/i,
          priority: -10
        }
      },
    },
  },
};
```
这两个配置组能帮助我们：

  * 将所有 node_modules 中的资源单独打包到 vendors-xxx-xx.js 命名的产物
  * 对引用次数大于等于 2 的模块 —— 也就是被多个 Chunk 引用的模块，单独打包

### 代码压缩

#### 使用 TerserWebpackPlugin 压缩 JS

Webpack5.0 后默认使用 Terser 作为 JavaScript 代码压缩器，简单用法只需通过 optimization.minimize 配置项开启压缩功能即可：

```
module.exports = {
  //...
  optimization: {
    minimize: true
  }
};
```

#### 使用 CssMinimizerWebpackPlugin 压缩 CSS

Webpack 社区中有不少实现 CSS 代码压缩的插件，例如：css-minimizer-webpack-plugin，用法：
```
const CssMinimizerPlugin = require("css-minimizer-webpack-plugin");
const MiniCssExtractPlugin = require("mini-css-extract-plugin");

module.exports = {
  //...
  module: {
    rules: [
      {
        test: /.css$/,
        // 注意，这里用的是 `MiniCssExtractPlugin.loader` 而不是 `style-loader`
        use: [MiniCssExtractPlugin.loader, "css-loader"],
      },
    ],
  },
  optimization: {
    minimize: true,
    minimizer: [
      // Webpack5 之后，约定使用 `'...'` 字面量保留默认 `minimizer` 配置
      "...",
      new CssMinimizerPlugin(),
    ],
  },
  // 需要使用 `mini-css-extract-plugin` 将 CSS 代码抽取为单独文件
  // 才能命中 `css-minimizer-webpack-plugin` 默认的 `test` 规则
  plugins: [new MiniCssExtractPlugin()],
};
```
这里的配置逻辑，一是使用 mini-css-extract-plugin 将 CSS 代码抽取为单独的 CSS 产物文件，这样才能命中 css-minimizer-webpack-plugin 默认的 test 逻辑；二是使用 css-minimizer-webpack-plugin 压缩 CSS 代码

#### 使用 HtmlMinifierTerser 压缩 HTML

现代 Web 应用大多会选择使用 React、Vue 等 MVVM 框架，这衍生出来的一个副作用是原生 HTML 的开发需求越来越少，HTML 代码占比越来越低，所以大多数现代 Web 项目中其实并不需要考虑为 HTML 配置代码压缩工作流。不过凡事都有例外，某些场景如 SSG 或官网一类偏静态的应用中就存在大量可被优化的 HTML 代码，为此社区也提供了一些相关的工程化工具，例如 html-minifier-terser。

html-minifier-terser 是一个基于 JavaScript 实现的、高度可配置的 HTML 压缩器，支持一系列 压缩特性 如：

  * collapseWhitespace：删除节点间的空字符串，如：
  ```
  <!-- 原始代码： -->
<div> <p>    foo </p>    </div>
<!-- 经过压缩的代码： -->
<div><p>foo</p></div>
  ```
  * removeComments：删除备注，如：
  ```
  <!-- 原始代码： -->
<!-- some comment --><p>blah</p>

<!-- 经过压缩的代码： -->
<p>blah</p>
  ```
我们可以借助 html-minimizer-webpack-plugin 插件接入 html-minifier-terser 压缩器

```
const HtmlWebpackPlugin = require("html-webpack-plugin");
const HtmlMinimizerPlugin = require("html-minimizer-webpack-plugin");

module.exports = {
  // ...
  optimization: {
    minimize: true,
    minimizer: [
      // Webpack5 之后，约定使用 `'...'` 字面量保留默认 `minimizer` 配置
      "...",
      new HtmlMinimizerPlugin({
        minimizerOptions: {
          // 折叠 Boolean 型属性
          collapseBooleanAttributes: true,
          // 使用精简 `doctype` 定义
          useShortDoctype: true,
          // ...
        },
      }),
    ],
  },
  plugins: [
    // 简单起见，这里我们使用 `html-webpack-plugin` 自动生成 HTML 演示文件
    new HtmlWebpackPlugin({
      templateContent: `<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
    <html>
      <head>
        <meta charset="UTF-8" />
        <title>webpack App</title>
      </head>
      <body>
        <input readonly="readonly"/>
        <!-- comments -->
        <script src="index_bundle.js"></script>
      </body>
    </html>`,
    }),
  ],
};
```

### 使用动态加载，减少首屏资源加载量；

Webpack 默认会将同一个 Entry 下的所有模块全部打包成一个产物文件 —— 包括那些与页面 关键渲染路径 无关的代码，这会导致页面初始化时需要花费多余时间去下载这部分暂时用不上的代码，影响首屏渲染性能，例如：

```
import someBigMethod from "./someBigMethod";

document.getElementById("someButton").addEventListener("click", () => {
  someBigMethod();
});
```

逻辑上，直到点击页面的 someButton 按钮时才会调用 someBigMethod 方法，因此这部分代码没必要出现在首屏资源列表中，此时我们可以使用 Webpack 的动态加载功能将该模块更改为异步导入，修改上述代码：

```
document.getElementById("someButton").addEventListener("click", async () => {
  // 使用 `import("module")` 动态加载模块
  const someBigMethod = await import("./someBigMethod");
  someBigMethod();
});
```

此时，重新构建将产生额外的产物文件 src_someBigMethod_js.js，这个文件直到执行 import 语句时 —— 也就是上例 someButton 被点击时才被加载到浏览器，也就不会影响到关键渲染路径了。

动态加载是 Webpack 内置能力之一，我们不需要做任何额外配置就可以通过动态导入语句(import、require.ensure)轻易实现。但请 注意，这一特性有时候反而会带来一些新的性能问题：一是过度使用会使产物变得过度细碎，产物文件过多，运行时 HTTP 通讯次数也会变多，在 HTTP 1.x 环境下这可能反而会降低网络性能，得不偿失；二是使用时 Webpack 需要在客户端注入一大段用于支持动态加载特性的 Runtime

因此，请务必慎重，多数情况下我们没必要为小模块使用动态加载能力！目前社区比较常见的用法是配合 SPA 的前端路由能力实现页面级别的动态加载，例如在 Vue 中：

```
import { createRouter, createWebHashHistory } from "vue-router";

const Home = () => import("./Home.vue");
const Foo = () => import(/* webpackChunkName: "sub-pages" */ "./Foo.vue");
const Bar = () => import(/* webpackChunkName: "sub-pages" */ "./Bar.vue");

// 基础页面
const routes = [
  { path: "/bar", name: "Bar", component: Bar },
  { path: "/foo", name: "Foo", component: Foo },
  { path: "/", name: "Home", component: Home },
];

const router = createRouter({
  history: createWebHashHistory(),
  routes,
});

export default router;
```

示例中，Home/Foo/Bar 三个组件均通过 import() 语句动态导入，这使得仅当页面切换到相应路由时才会加载对应组件代码。另外，Foo 与 Bar 组件的导入语句比较特殊：

```
import(/* webpackChunkName: "sub-pages" */ "./Bar.vue");
```
webpackChunkName 用于指定该异步模块的 Chunk 名称，相同 Chunk 名称的模块最终会打包在一起，这一特性能帮助开发者将一些关联度较高，或比较细碎的模块合并到同一个产物文件，能够用于管理最终产物数量。

### HTTP 缓存优化

注意，Webpack 只是一个工程化构建工具，没有能力决定应用最终在网络分发时的缓存规则，但我们可以调整产物文件的名称(通过 Hash)与内容(通过 Code Splitting)，使其更适配 HTTP 持久化缓存策略。

Webpack 提供了一种模板字符串(Template String)能力，用于根据构建情况动态拼接产物文件名称(output.filename)，规则稍微有点复杂，但从性能角度看，比较值得关注的是其中的几个 Hash 占位符，包括：

 * [fullhash]：整个项目的内容 Hash 值，项目中任意模块变化都会产生新的 fullhash；
 * [chunkhash]：产物对应 Chunk 的 Hash，Chunk 中任意模块变化都会产生新的 chunkhash；
 * [contenthash]：产物内容 Hash 值，仅当产物内容发生变化时才会产生新的 contenthash，因此实用性较高。

用法很简单，只需要在 output.filename 值中插入相应占位符即可，如 "[name]-[contenthash].js"。我们来看个完整例子，假设对于下述源码结构：

```
src/
├── index.css
├── index.js
└── foo.js
```
之后，使用下述配置：

```
module.exports = {
  // ...
  entry: { index: "./src/index.js", foo: "./src/foo.js" },
  output: {
    filename: "[name]-[contenthash].js",
    path: path.resolve(__dirname, "dist"),
  },
  plugins: [new MiniCssExtractPlugin({ filename: "[name]-[contenthash].css" })],
};
```

构建后每个产物文件名都会带上一段由产物内容计算出的唯一 Hash 值，文件内容不变，Hash 也不会变化，这就很适合用作 HTTP 持久缓存 资源：

```
# HTTP Response header

Cache-Control: max-age=31536000
```
此时，产物文件不会被重复下载，一直到文件内容发生变化，引起 Hash 变化生成不同 URL 路径之后，才需要请求新的资源文件，能有效提升网络性能，因此，生产环境下应尽量使用 [contenthash] 生成有版本意义的文件名。

### 使用外置依赖

设想一个场景，假如我们手头上有 10 个用 Vue 构建的 SPA 应用，这 10 个应用都需要各自安装、打包、部署、分发同一套相似的 Vue 基础依赖，最终用户在访问这些应用时也需要重复加载相同基础包代码，那有没有办法节省这部分流量呢？有 —— 使用 Webpack 的 externals 特性。

externals 的主要作用是将部分模块排除在 Webpack 打包系统之外，例如：

```
module.exports = {
  // ...
  externals: {
    lodash: "_",
    echarts: "echarts",
    vue: "Vue",
  },
};
```

使用上述配置后，Webpack 会 预设 运行环境中已经内置 Lodash,Vue, echarts 库 —— 无论是通过 CDN 还是其它方式注入，所以不需要再将这些模块打包到产物中

虽然结果上看浏览器还是得消耗这部分流量，但结合 CDN 系统特性，一是能够就近获取资源，缩短网络通讯链路；二是能够将资源分发任务前置到节点服务器，减轻原服务器 QPS 负担；三是用户访问不同站点能共享同一份 CDN 资源副本。所以网络性能效果往往会比重复打包好很多。

### 使用 Tree-Shaking 删除多余模块导出

Tree-Shaking 是一种基于 ES Module 规范的 Dead Code Elimination 技术，它会在运行过程中静态分析模块之间的导入导出，判断哪些模块导出值没有被其它模块使用 —— 相当于模块层面的 Dead Code，并将其删除。

在 Webpack 中，启动 Tree Shaking 功能必须同时满足两个条件：

 * 配置 optimization.usedExports 为 true，标记模块导入导出列表；
 * 启动代码优化功能，可以通过如下方式实现：
    - 配置 mode = production
    - 配置 optimization.minimize = true
    - 提供 optimization.minimizer 数组

```
// webpack.config.js
module.exports = {
  mode: "production",
  optimization: {
    usedExports: true,
  },
};
```

之后，Webpack 会对所有使用 ESM 方案的模块启动 Tree-Shaking，例如对于下面的代码：

```
// index.js
import {bar} from './bar';
console.log(bar);

// bar.js
export const bar = 'bar';
export const foo = 'foo';
```

bar.js 模块导出了 bar 、foo ，但只有 bar 值被 index 模块使用，经过 Tree Shaking 处理后，foo 变量会被视作无用代码删除，最终有效的代码结构：

```
// index.js
import {bar} from './bar';
console.log(bar);

// bar.js
export const bar = 'bar';
```

### 使用 Scope Hoisting 合并模块

默认情况下 Webpack 会将模块打包成一个个单独的函数，例如：

```
// common.js
export default "common";

// index.js
import common from './common';
console.log(common);
```

经过 Webpack 打包后会生成：

```
"./src/common.js":
  ((__unused_webpack_module, __webpack_exports__, __webpack_require__) => {
     const __WEBPACK_DEFAULT_EXPORT__ = ("common");
     __webpack_require__.d(__webpack_exports__, {
      /* harmony export */
      "default": () => (__WEBPACK_DEFAULT_EXPORT__)
      /* harmony export */
    });
  }),
"./src/index.js":
  ((__unused_webpack_module, __webpack_exports__, __webpack_require__) => {
      var _common__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__( /*! ./common */ "./src/common.js");
      console.log(_common__WEBPACK_IMPORTED_MODULE_0__)
  })
```

这种处理方式需要将每一个模块都包裹进一段相似的函数模板代码中，好看是好看，但浪费网络流量啊。为此，Webpack 提供了 Scope Hoisting 功能，用于 将符合条件的多个模块合并到同一个函数空间 中，从而减少产物体积，优化性能。例如上述示例经过 Scope Hoisting 优化后，生成代码：

```
((__unused_webpack_module, __webpack_exports__, __webpack_require__) => {
    ;// CONCATENATED MODULE: ./src/common.js
    /* harmony default export */ const common = ("common");
    
    ;// CONCATENATED MODULE: ./src/index.js
    console.log(common);
})
```

Webpack 提供了三种开启 Scope Hoisting 的方法：

 * 使用 mode = 'production' 开启生产模式；
 * 使用 optimization.concatenateModules 配置项；
 * 直接使用 ModuleConcatenationPlugin 插件。

### 监控产物体积

Webpack 专门为此提供了一套 性能监控方案，当构建生成的产物体积超过阈值时抛出异常警告，以此帮助我们时刻关注资源体积，避免因项目迭代增长带来过大的网络传输，用法：

```
module.exports = {
  // ...
  performance: {    
    // 设置所有产物体积阈值
    maxAssetSize: 172 * 1024,
    // 设置 entry 产物体积阈值
    maxEntrypointSize: 244 * 1024,
    // 报错方式，支持 `error` | `warning` | false
    hints: "error",
    // 过滤需要监控的文件类型
    assetFilter: function (assetFilename) {
      return assetFilename.endsWith(".js");
    },
  },
};
```

