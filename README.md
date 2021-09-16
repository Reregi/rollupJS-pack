# rollupJS_pack 基于 rollupJS 的库打包

[toc]

## 一、理清项目依赖

### 1.1 rollup 有关

> npm i -D rollup rollup-plugin-babel rollup-plugin-postcss rollup-plugin-pug rollup-plugin-terser rollup-plugin-typescript2 rollup-plugin-vue

**根据项目添加需要用到的插件**

| 依赖名                    | 作用                                                    |
| ------------------------- | ------------------------------------------------------- |
| rollup                    | rollup 核心文件用于打包                                 |
| rollup-plugin-babel       | rollup 的 babel 插件，用于将 es6 转 es5                 |
| rollup-plugin-postcss     | rollup 的 postcss 插件，用于将 css 代码封装进 js 文件里 |
| rollup-plugin-pug         | rollup 的 pug 模板插件，用于将 pug 代码转成 html        |
| rollup-plugin-terser      | rollup 的代码压缩插件                                   |
| rollup-plugin-typescript2 | rollup 的 typescript 插件                               |
| rollup-plugin-vue         | rollup 的 vue 插件,注意版本对 vue3.0 的支持             |

### 1.2 插件有关

> npm i -D @babel/core @babel/preset-env @babel/preset-stage-2 @babel/plugin-transform-runtime @babel/plugin-external-helpers @babel/register

#### 1.2.1 babel

| 依赖名                          | 作用                                                                                               |
| ------------------------------- | -------------------------------------------------------------------------------------------------- |
| @babel/core                     | babel 核心文件                                                                                     |
| @babel/preset-env               | 可以根据配置的目标浏览器或者运行环境来自动将 ES2015+的代码转换为 es5                               |
| @babel/preset-stage-2           | 包含不同的转码插件                                                                                 |
| @babel/plugin-transform-runtime | 转译新增的 API 与全局对象                                                                          |
| @babel/plugin-external-helpers  | 避免重复引用的插件                                                                                 |
| @babel/register                 | 引入 @babel/register 之后所有 require 并以.es6, .es, .jsx 和 .js 为后缀的模块都会经过 babel 的转译 |

#### 1.2.2 css 预处理器

> npm i -D scss

| 依赖名 | 作用           |
| ------ | -------------- |
| scss   | 编译 scss 代码 |

#### 1.2.3 vue+pug+typescript 框架模板依赖

> npm i -D vue typescript @vue/compiler-sfc pug vue-dts-gen

| 依赖名            | 作用                             |
| ----------------- | -------------------------------- |
| vue               | vue 核心代码                     |
| typescript        | typescript 编译                  |
| @vue/compiler-sfc | sfc 模板解析                     |
| pug               | pug 模板编译                     |
| vue-dts-gen       | 用于`*.vue`文件生成声明文件 d.ts |

### 1.3 依赖配置文件

**rollup.config.js - rollup 配置文件**

```js
import vue from "rollup-plugin-vue";
import path from "path";
import { terser } from "rollup-plugin-terser";
import postcss from "rollup-plugin-postcss";
import ts from "rollup-plugin-typescript2";
import pug from "rollup-plugin-pug";
import babel from "rollup-plugin-babel";
const getPath = (_path) => path.resolve(__dirname, _path);

module.exports = [
  {
    // 入口
    input: "index.ts",
    // 出口
    output: [
      {
        //打包后的入口文件
        file: "lib/index.js",
        // 配置打包模块化的方式 es:ESM  cjs:CommonJS
        //es的打包格式 babel会转换部分es6以上的代码，不会转换成es5，原因是ESM就是es6标准
        format: "umd", //注意打包格式 umd是兼容全部模块化的
        globals: {
          vue: "vue", //指明global.vue 是外部依赖vue
        },
        exports: "named",
      },
    ],
    // 插件
    plugins: [
      vue({
        // 把单文件组件中的样式，插入到html中的style标签
        css: true,
        // 把组件转换成 render 函数
        compileTemplate: true,
      }),
      // 代码压缩
      terser(),
      postcss(),
      ts({
        tsconfig: getPath("./tsconfig.json"), // 导入本地ts配置
      }),
      pug(),
      babel({
        exclude: "node_modules/**", // 排除node_module下的所有文件
        runtimeHelpers: true,
        extensions: [".vue"],
      }),
    ],
  },
];
```

**.babelrc - babel 配置文件**

```json
{
  "presets": [
    [
      "@babel/preset-env",
      {
        "modules": false
      }
    ]
  ],
  "plugins": ["@babel/plugin-transform-runtime"]
}
```

**tsconfig.json - ts 配置文件**
**json 文件不能注释，下方为标记**

```json
{
  "compilerOptions": {
    "target": "esnext",
    "module": "esnext",
    "moduleResolution": "node",
    "strict": true,
    "jsx": "preserve",
    "sourceMap": true,
    "resolveJsonModule": true,
    "esModuleInterop": true,
    "lib": ["esnext", "dom"],
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"] //搭配vite项目的文件夹别名
    },
    "declaration": true, //关键生成d.ts声明文件
    "declarationDir": "./types" //生成声明文件的路径
  },
  "include": [
    "src/**/*.ts",
    "src/**/*.d.ts",
    "src/**/*.tsx",
    "src/**/*.vue",
    "*.vue"
  ]
}
```

## 二、构建

### 2.1 构建前准备工作

`rollup-plugin-typescript2`这个插件再打包后自动帮我们生成`d.ts`声明文件
但不幸的是，不知道为什么打包前读取我的 vue 文件的引用为 `找不到模块“./xxx.vue”或其相应的类型声明。`
报的是 TS 2307 的错。问题不大，生成声明文件提供下就行了。
所以使用`vue-dts-gen`来给 vue 文件来生成声明文件

**npm 安装**

> npm i -g vue-dts-gen

**使用方法**

> vue-dts-gen "src/\*.{ts,vue}" //根据路径匹配文件，生成的目录是根据 tsconfig.json 配置

需要的声明的文件是`demo.vue`那生成的对应声明文件就是`demo.d.ts`,但是当你复制到对应的 vue 文件的路径下的时候，打包还是报错，因为它需要的是(.vue).d.ts 的声明文件，所以将`demo.d.ts`改为`demo.vue.d.ts`，就解决了

**使用 PowerToy 的 PowerRename 批量重命名**
![PowerRename 重命名](https://s3.jpg.cm/2021/09/13/IYOTXy.png)

### 2.2 开始构建

**package.json 项目所需依赖 打包脚本 项目信息 用于打包后上传到 npm 展示的一些信息**

```json
{
  "name": "项目名",
  "version": "1.0.0",
  "description": "项目描述",
  "keywords": ["关键词"],
  "homepage": "项目的主页，可以是git仓库，也可以项目展示的页面",
  "repository": {
    "type": "git",
    "url": "git+https://github.com/xxx/xxx.git"
  },
  "author": "作者",
  "files": ["打包的文件", "打包的路径"], //npm会根据这个把打包的文件，路径下的文件，打包发布至npm
  "typings": "types/index.d.ts", //项目声明文件的路径
  "module": "lib/index.js", //库的入口地址
  "scripts": {
    "build": "rollup -c"
  },
  "devDependencies": {
    "@babel/core": "^7.15.5",
    "@babel/plugin-external-helpers": "^7.14.5",
    "@babel/plugin-transform-runtime": "^7.15.0",
    "@babel/preset-env": "^7.15.4",
    "@babel/register": "^7.15.3",
    "@vue/compiler-sfc": "^3.2.10",
    "pug": "^3.0.2",
    "rollup": "^2.56.3",
    "rollup-plugin-babel": "^4.4.0",
    "rollup-plugin-node-resolve": "^5.2.0",
    "rollup-plugin-postcss": "^4.0.1",
    "rollup-plugin-pug": "^1.1.1",
    "rollup-plugin-terser": "^7.0.2",
    "rollup-plugin-typescript2": "^0.30.0",
    "rollup-plugin-vue": "^6.0.0",
    "sass": "^1.39.0",
    "typescript": "^4.4.2",
    "vue": "^3.2.11",
    "vue-dts-gen": "^0.3.0"
  },
  "dependencies": {}
}
```

运行 npm 脚本

> npm run build

打包成功后根目录下会出现 lib 文件夹，里面有`index.js`,看看里面的内容，如果特别少，需要考虑是不是自己打包失败了，很有可能是因为打包前的引用关系错误，导致没有打包成功

### 2.3 构建后测试

打包成 npm 包

> npm pack
> 生成 xxxxx-版本号.tgz

项目测试

> npm install xxxxx-版本号.tgz

然后引用包，测试

```js
import xxx from "xxxxx"; //'xxxxx'是package.json里声明的name
```

### 2.4 测试成功后发布至 npm

#### 2.4.1 注册 npm 账号

方法一： [npm 官网注册](https://www.npmjs.com/signup)
方法二: 命令行注册 根据提示输入 name password e-mail

> npm adduser

#### 2.4.2 使用 npm 发布

国内用户平常用惯的淘宝镜像，发布前的登录失败，常常是因为下载源没有进行切换

> npm config get registry http://registry.npmjs.org

登录

> npm login

发布

> npm publish

首次的发布失败，应该是 name 重复了，不能有重复的包名。注意后续的版本发布，需要更换版本号。

按照自己的需求 更换淘宝镜像

> npm config set registry https://registry.npm.taobao.org

#### 2.5 发布后测试

先把之前包安装的卸载了，后续 npm 安装

> npm uninstall xxxxx
> npm install xxxxx

引用测试。构建打包测试成功后，npm 发布的包一般都没有什么问题

## 三、问题一览

### 3.1 依赖是全部都需要吗？

看着自己项目来配依赖，比如 pug 什么的，你写的时候不需要模板语法，只写 html 那就不需要，css 预处理器，也可以试其他的，项目框架是 react 的就换 react 的插件。

### 3.2 [!] (plugin vue) Error: Error parsing JavaScript expression: Unexpected token (1:3)

该问题 大多数是 vue 文件里的模板语法导致的，换回 html 没有报错。
我把组件单独拿出来打包不会出现这个状况，但是在 vite 项目使用 pug 目标你语法内打包组件会出现这个。

### 3.3 后续问题

....之后补充
