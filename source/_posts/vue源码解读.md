# vue 源码阅读

## vue 源码文件说明

- script:包含与构建相关的脚本和配置文件
  - script/alias.js:再所有源代码和测试中使用的模块导入别名
  - script/config.js:项目的构建配置
- dist:包含用于分发的内置文件。构建后文件的输出目录
- build:构建相关的文件，一般情况下不需要动
- flow:JS 静态类型检查工具[Flow](https://flowtype.org/)的类型声明
- examples:存放一些使用 vue 开发的应用案例
- packages:包含 vue-server-renderer 和 vue-template-compiler，作为独立的 NPM 软件包开发。他们从源代码生成的，并且始终与主 vue 软件包具有相同的版本
- test:包含所有的测试代码
- src:包含源代码
  - compiler:包含用于模板到渲染功能的编译器的代码
    - parser:存放将模板字符串转换成元素抽象语法树的代码(将模板字符串转换成元素 AST)
    - codegen:存放从抽象语法树(AST)生成 render 函数的代码,减少代码大小
    - optimizer.js:分析静态树,优化 vdom 渲染
  - core:存放通用的，平台无关的运行时代码
    - observer:响应式实现，包含数据观测的核心代码
    - vDom:虚拟 Dom 的 creation 和 patching 的代码
    - instance:包含 vue 实例构造函数和原型方法
    - global-api:包含 Vue 全局 API
    - components:包含通用抽象组件
  - server:包含与服务器端渲染有关的代码
  - platforms:包含平台特定的代码
    - weex:weex 平台支持
    - web:web 平台支持
      - entry-runtime.js:运行时构建的入口
      - entry-runtime-with-compiler.js:独立构建版本的入口
      - entry-compiler.js:vue-template-compiler 包的入口文件
      - entry-server-renderer.js:vue-server-renderer 包的入口文件
  - sfc:包含单个文件组件(\*.vue 文件)的解析逻辑。在 vue-template-compiler 包装中使用
  - shared:包含在整个代码库中共享的实用程序
  - types:包含 typescript 类型定义
    - test:包含类型定义测试

### 重要的目录

- compiler:编译，用来将 template 转化为 render 函数
- core:vue 的核心代码，包含响应式实现，虚拟 DOM，vue 实例方法的挂载，全局方法，抽象出来的通用组件等
- platform:不同平台的入口文件，主要是 web 平台和 weex 平台的，不同平台有其特殊的构建过程，重点是 web 平台
- server:服务端渲染(SSR)的相关代码，SSR 主要把组件直接渲染为 HTML 并由 Server 端直接提供给 client 端
- sfc:主要是.vue 文件解析的逻辑
- shared:一些通用的工具方法，有一些是为了增加代码可读性而设置的
- 通过 src/platforms/web/entry-runtime.js 文件作为运行时构建的入口，ESM 方式输出 dist/vue.runtime.esm.js,CJS 方式输出 dist/vue.runtime.common.js,UMD 方式输入 dist/vue.runtime.js,不包含模板 template 到 render 函数的编译器
- 通过 src/platforms/web/entry-runtime-with-compiler 文件作为运行时构建的入口，ESM 方式输出 dist/vue.esm.js,CJS 方式输出 dist/vue.common.js,UMD 方式输入 dist/vue.js.包含 compiler，包含 template 到 render 函数的编译器

### 分析流程

- package.json 文件

```
{
  "name": "vue",
  "version": "2.6.11",
  "description": "Reactive, component-oriented view layer for modern web interfaces.",
  "main": "dist/vue.runtime.common.js",
  "module": "dist/vue.runtime.esm.js",
  "unpkg": "dist/vue.js",
  "jsdelivr": "dist/vue.js",
  "typings": "types/index.d.ts",
  "files": [
    "src",
    "dist/*.js",
    "types/*.d.ts"
  ],
  "sideEffects": false,
  "scripts": {
    //构建完整umd模块vue
    "dev": "rollup -w -c scripts/config.js --sourcemap --environment TARGET:web-full-dev",
    // 构建运行时cjs模块的vue
    "dev:cjs": "rollup -w -c scripts/config.js --environment TARGET:web-runtime-cjs-dev",
    //构建运行时es模块的vue
    "dev:esm": "rollup -w -c scripts/config.js --environment TARGET:web-runtime-esm",
    //构建测试版的vue
    "dev:test": "karma start test/unit/karma.dev.config.js",
    //构建web-server-renderer包
    "dev:ssr": "rollup -w -c scripts/config.js --environment TARGET:web-server-renderer",
    //构建compiler包
    "dev:compiler": "rollup -w -c scripts/config.js --environment TARGET:web-compiler ",
    "dev:weex": "rollup -w -c scripts/config.js --environment TARGET:weex-framework",
    "dev:weex:factory": "rollup -w -c scripts/config.js --environment TARGET:weex-factory",
    "dev:weex:compiler": "rollup -w -c scripts/config.js --environment TARGET:weex-compiler ",
    "build": "node scripts/build.js",
    "build:ssr": "npm run build -- web-runtime-cjs,web-server-renderer",
    "build:weex": "npm run build -- weex",
    "test": "npm run lint && flow check && npm run test:types && npm run test:cover && npm run test:e2e -- --env phantomjs && npm run test:ssr && npm run test:weex",
    "test:unit": "karma start test/unit/karma.unit.config.js",
    "test:cover": "karma start test/unit/karma.cover.config.js",
    "test:e2e": "npm run build -- web-full-prod,web-server-basic-renderer && node test/e2e/runner.js",
    "test:weex": "npm run build:weex && jasmine JASMINE_CONFIG_PATH=test/weex/jasmine.js",
    "test:ssr": "npm run build:ssr && jasmine JASMINE_CONFIG_PATH=test/ssr/jasmine.js",
    "test:sauce": "npm run sauce -- 0 && npm run sauce -- 1 && npm run sauce -- 2",
    "test:types": "tsc -p ./types/test/tsconfig.json",
    "lint": "eslint src scripts test",
    "flow": "flow check",
    "sauce": "karma start test/unit/karma.sauce.config.js",
    "bench:ssr": "npm run build:ssr && node benchmarks/ssr/renderToString.js && node benchmarks/ssr/renderToStream.js",
    "release": "bash scripts/release.sh",
    "release:weex": "bash scripts/release-weex.sh",
    "release:note": "node scripts/gen-release-note.js",
    "commit": "git-cz"
  },
  "gitHooks": {
    "pre-commit": "lint-staged",
    "commit-msg": "node scripts/verify-commit-msg.js"
  },
  "lint-staged": {
    "*.js": [
      "eslint --fix",
      "git add"
    ]
  },
  "repository": {
    "type": "git",
    "url": "git+https://github.com/vuejs/vue.git"
  },
  "keywords": [
    "vue"
  ],
  "author": "Evan You",
  "license": "MIT",
  "bugs": {
    "url": "https://github.com/vuejs/vue/issues"
  },
  "homepage": "https://github.com/vuejs/vue#readme",
  "config": {
    "commitizen": {
      "path": "./node_modules/cz-conventional-changelog"
    }
  }
}
```

- umd 支持外链规范的文件输出，次文件可以直接使用 script 标签
- cjs:commonJS Module，遵循 commonJS Module 规范的文件输出
- es:ES Modules，使用 ES6 的模板语法输出
- amd:AMD Module 遵循 AMD Module 规范的文件输出
- 完整版:compiler+runtime，同时包含编译器和运行时版本就是完整版

```
 new Vue({
   template:'<div>{{hi}}</div>'
 })
```

完整版本可以直接写在 HTML 或 template 中查看视图效果。用 webpack 引入。需要配置 alias,

- 运行时版:不包含模板编译器

```

new Vue({
  el:'#app',
  data:{
    value:0
  },
  render(h){
    return h('div',this.n)
  }
})
```

h 是 vue.runtime.js 提供的函数，他接受模板字符串中的参数，返回渲染好的原始的 html

- 入口文件 scripts/config.js
  - 通过这个文件我们可以看到所有打包模式的 vue 的文件入口的文件，当前主要查看的是完整版，通过 package.json 中 dev 命令中的 web-full-dev 来查找 config.js 中 web-full-dev 对应的文件是 web/entry-runtime-with-compiler.js，打包之后的文件是 dest
  - 为了能在浏览器中查到格式化的 vue 源码， 我们可以在 package.json 的 dev 命令中加上--sourcemap
  - src/platforms/web/entry-runtime-with-compiler.js
  - import Vue from './runtime/index' src/platforms/web/runtime/index
  - import Vue from "core/index" src/core/index.js
  - import Vue from './instance/index' src/core/instance/index.js
    通过这个可以找到对应的入口文件

```
import { initMixin } from './init'
import { stateMixin } from './state'
import { renderMixin } from './render'
import { eventsMixin } from './events'
import { lifecycleMixin } from './lifecycle'
import { warn } from '../util/index'

function Vue (options) {
if (process.env.NODE_ENV !== 'production' &&
!(this instanceof Vue)
) {
warn('Vue is a constructor and should be called with the `new` keyword')
}
this._init(options)
}

initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)

export default Vue

```

在这个文件中创建了一个 VUE 构造函数,然后通过 5 个方法来处理这个构造函数
