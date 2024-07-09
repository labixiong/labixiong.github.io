---
layout:     post
title:      qiankunjs微前端框架详解
subtitle:   qiankunjs微前端框架并实践，搭建过程中出现什么样的问题
date:       2024-03-25
author:     LBX
header-img: img/bg-little-universe.jpg
catalog: true
tags:
    - 前端
    - qiankunjs
    - vue2
    - vue3
    - wepack
    - vite
---

> qiankun 是一个基于single-spa的微前端实现库，旨在帮助大家能更简单的构建一个生产可用微前端架构系统。

# 简介

qiankun 是一个基于 [single-spa](https://github.com/CanopyTax/single-spa) 的[微前端](https://micro-frontends.org/)实现库，旨在帮助大家能更简单的构建一个生产可用微前端架构系统。

qiankun 孵化自蚂蚁金融科技基于微前端架构的云产品统一接入平台。

## qiankun的核心设计概念


- 简单 

    由于主应用微应用都能做到技术栈无关，qiankun 对于用户而言只是一个类似 jQuery 的库，你需要调用几个 qiankun 的 API 即可完成应用的微前端改造。同时由于 qiankun 的 HTML entry 及沙箱的设计，使得微应用的接入像使用 iframe 一样简单。
    
- 解耦/技术栈无关

    微前端的核心目标是将巨石应用拆解成若干可以自治的松耦合微应用，而 qiankun 的诸多设计均是秉持这一原则，如 HTML entry、沙箱、应用间通信等。这样才能确保微应用真正具备 独立开发、独立运行 的能力。
    
## 为什么不是iframe？

如果不考虑体验问题，iframe是微前端的最优解

iframe 最大的特性就是提供了浏览器原生的硬隔离方案，不论是样式隔离、js 隔离这类问题统统都能被完美解决。但他的最大问题也在于他的隔离性无法被突破，导致应用间上下文无法被共享，随之带来的开发体验、产品体验的问题。

## qiankun特性

-    **基于 [single-spa](https://github.com/CanopyTax/single-spa)** 封装，提供了更加开箱即用的 API。
-    **技术栈无关**，任意技术栈的应用均可 使用/接入，不论是 React/Vue/Angular/JQuery 还是其他等框架。
-   **HTML Entry 接入方式**，让你接入微应用像使用 iframe 一样简单。
-   **样式隔离**，确保微应用之间样式互相不干扰。
-   **JS 沙箱**，确保微应用之间 全局变量/事件 不冲突。
-   **资源预加载**，在浏览器空闲时间预加载未打开的微应用资源，加速微应用打开速度。
-   **umi 插件**，提供了 [@umijs/plugin-qiankun](https://github.com/umijs/plugins/tree/master/packages/plugin-qiankun) 供 umi 应用一键切换成微前端架构系统。

## 什么是微前端？

微前端是一种多个团队通过独立发布功能的方式来共同构建现代化 web 应用的技术手段及方法策略。

微前端架构旨在解决单体应用在一个相对长的时间跨度下，由于参与的人员、团队的增多、变迁，从一个普通应用演变成一个巨石应用后，随之而来的应用不可维护的问题。这类问题在企业级 Web 应用中尤其常见。

## 微前端的核心价值

1. 技术栈无关 - 主框架不限制接入应用的技术栈，微应用具备完全自主权
2. 独立开发、独立部署 - 微应用仓库独立，前后端可独立开发，部署完成后主框架自动完成同步更新
3. 增量升级 - 在面对各种复杂场景时，我们通常很难对一个已经存在的系统做全量的技术栈升级或重构，而微前端是一种非常好的实施渐进式重构的手段和策略
4. 独立运行时 - 每个微应用之间状态隔离，运行时状态不共享


# 如何快速上手

## 安装qiankun

```shell
yarn add qiankun # 或者npm i qiankun -S
```

## 在主应用注册子应用

```javascript
// 需要在主应用入口函数中进行注册
import { registerMicroApps, start } from 'qiankun';


registerMicroApps([
  {
    name: 'react app', // app name registered
    entry: '//localhost:7100',
    container: '#yourContainer',
    activeRule: '/yourActiveRule',
  },
  {
    name: 'vue app',
    entry: { scripts: ['//localhost:7100/main.js'] },
    container: '#yourContainer2',
    activeRule: '/yourActiveRule2',
  },
], {
  beforeLoad: app => {},
  beforeMount: app => {
    // 子应用挂载前操作，可以将当前的子应用的配置保存起来
  },
  afterUnmount: app => {
    // 子应用卸载后清除当前子应用的配置等等
  },
});


start();


// 子应用入口文件代码
export function bootstrap() {}

export function mount() {}

export function unmount() {}
```

- **name:** 微应用名字 需确保唯一性
- **entry：** 微应用入口路径
- **activeRule：** 注册微应用时，微应用的路由匹配规则

注册成功后，当浏览器的url发生变化时，会自动触发qiankun的匹配逻辑，所有 activeRule 规则匹配上的微应用就会被插入到指定的 container 中，同时依次调用微应用暴露出的生命周期钩子。

如果微应用不是直接跟路由关联的时候，你也可以选择手动加载微应用的方式：

```javascript
// 主应用
import { loadMicroApp } from 'qiankun';

loadMicroApp({
  name: 'app',
  entry: '//localhost:7100',
  container: '#yourContainer',
  props: {}
}, {
   sandbox: true,
   // ...
});


// 子应用入口文件中除了要导出bootstrap、mount、unmount还需要额外导出一个update生命周期函数

// ...
export async function update(props) {
    // ...
    render(props)
}

```

name、entry、container三个属性为必填项,props为可选属性，在初始化时主应用需要传递给微应用的数据。

### **自动加载和手动加载的区别：**

1. 自动加载的方式增加了 `activeRule` 配置，此属性是必选的，为微应用的激活规则
2. 自动加载的第二个参数是全局的微应用生命周期钩子；手动加载的第二个参数是一些配置，包括sandbox(是否开启沙箱，默认为开启)、singular（是否为单实例场景，指同一时间只会渲染一个微应用，默认为false）等属性
3. 手动加载微应用会返回微应用的实例，包括但不限于微应用的生命周期钩子，可以保存起来供主应用在合适的位置进行调用

[loadMicroApp API](https://qiankun.umijs.org/zh/api#loadmicroappapp-configuration)

### 手动加载微应用的运用场景

如果微应用是一个不带路由的可独立运行的业务组件，可以使用手动加载这个应用


## 接入微应用

微应用无需安装qiankun，即可使用，但需要微应用在项目入口文件中暴露指定生命周期函数

### 导出相应的生命周期钩子

```javascript
// 只会在微应用初始化的时候调用一次，下次再进入会直接跳过该函数触发mount函数
// 通常做一些初始化操作，对微应用进行预设，还有不会在unmount阶段被销毁的应用级别的缓存等等
export async function bootstrap(props) {}

// 每次进入都会调用mount方法，通常在这里触发渲染方法，开始渲染页面
export async function mount(props) {
    render(props)
}

// 在此处卸载微应用的实例、清空路由、清空html等操作
export async function unmount() {}

// 可选生命周期钩子，仅使用loadMicroApp方式手动加载微应用时生效
export async function update(props) {}
```


### 打包配置

以webpack为例

```javascript
const packageName = require('./package.json').name;


module.exports = {
  output: {
    library: `${packageName}-[name]`,
    libraryTarget: 'umd',
    
    // chunkLoadingGlobal为webpack v5的配置
    // 如使用的是webpack v4版本，需要将此配置改为jsonpFunction，属性值与webpack v5版本一致
    // webpack v4 -- jsonpFunction: `webpackJsonp_${packageName}`
    chunkLoadingGlobal: `webpackJsonp_${packageName}`,
  },
};
```

**library** - ```${pkgson.name}-[name]```,这里的`pkgson.name`值是`vue2Template`会将你的library bundle暴露为名为`vue2Template-name`的全局变量，使用者会通过此名称来import。 

**libraryTarget** - 'umd'，通常与`library`选项配合使用，最简单的解释就是以umd的方式对library进行暴露，方便在AMD或者CommonJS require之后使用。

**jsonpFunction** - `webpackJsonp_${pkgson.name}`，用于异步加载模块或连接多个初始化模块，如果使用了`output.library`这个配置，这个library name会自动拼接`output.jsonpFunction`的值，所以这里需要手动指定。

**chunkLoadingGlobal** - 用于`webpack`加载`chunks`

### 新增 `public-path` 文件

项目中新增 `public-path` 文件，用于修改运行时的 `publicPath`

```javascript
// 例如增加到src目录下，src/public-path.js文件内容
if (window.__POWERED_BY_QIANKUN__) {
  __webpack_public_path__ = window.__INJECTED_PUBLIC_PATH_BY_QIANKUN__
}

// 在入口文件中引入
import './public-path'
// ...
```

### 微应用设置路由

路由模式建议使用 `history` 模式的路由，路由的 `base` 值和它的 `activeRule` 值保持一致

### 配置允许跨域

同样是以webpack为例：

```javascript
// webpack.config.js
module.exports = {
    // ...
    devServer: {
        // ...
        headers: {
          'Access-Control-Allow-Origin': '*'
        }
    }
}
```


# 常见问题

[qiankun官网整理的常见问题](https://qiankun.umijs.org/zh/faq)

## 主应用和子应用之间如何进行通信

https://blog.csdn.net/weixin_43972437/article/details/128154083

https://www.cnblogs.com/goloving/p/15599561.html

1. `localStorage`、`sessionStorage`

    主应用通过 `localStorage` 设置全局主题和中英文模式,以便微应用中使用

2. `Actions`
    `qiankun`内部提供了 `initGlobalState` 方法用于注册 `MicroAppStateActions` 实例用于通信，该实例有三个方法:
    - `setGlobalState`: 全局状态设置新的值时，通知所有观察者函数
    - `onGlobalStateChange`: 注册观察者函数，当全局状态发生变化时触发观察者函数
    - `offGlobalStateChange`: 移除当前应用的状态监听，微应用 umount 时会默认调用
    ```javascript
    // 主应用
    import { initGlobalState, MicroAppStateActions } from 'qiankun';
    
    // 初始化 state
    const actions: MicroAppStateActions = initGlobalState(state);

    actions.onGlobalStateChange((state, prev) => {
      // state: 变更后的状态; prev 变更前的状态
      console.log(state, prev);
    });
    actions.setGlobalState(state);
    actions.offGlobalStateChange();
    
    // 微应用
    // 在生命周期mount中获取通信方法，使用方式和主应用一致
    export function mount(props) {
        props.onGlobalStateChange((state, prev) => {
            // state: 变更后的状态; prev 变更前的状态
            console.log(state, prev);
        })
    }
    ```
    
    此方式使用简单，官方支持性高，适合通信较少的业务场景，例如：主应用进行布局样式切换、中英文切换、主题切换、切换用户所属单位、组织等，微应用根据当前的全局状态进行更改
   
    缺点就是微应用单独运行时需要额外配置没有 `Actions` 的逻辑，通信场景较多时，容易出现状态混乱、维护困难等问题
4. `Shared`
    
    主应用通过 `vuex` `redux` `mobx`等任一状态管理工具正常维护一个状态，然后创建一个 `Shared`实例，遵循开闭原则，通过 `props` 将 `Shared` 实例传递给微应用。
    
    同样的微应用也需要维护一个 `Shared`, 当在主应用下运行时，就对当前维护的 `Shared` 进行重载，如果单独运行时就会使用自身的 `Shared`
    
    微应用可以自由选择状态管理库，无需了解主应用的状态池细节，同时也将具备独立运行的能力。
    
   缺点也就是主应用和微应用都需要维护 `Shared`,增加了维护成本和项目复杂度
   
   适用于主应用和微应用间频繁交互的场景中
5. `props`
    
    主应用在注册微应用时通过 `props` 属性传递给微应用所需要的属性

## 主应用和子应用公共组件、公共样式，这些如何进行维护？

**公共组件：**

官网解释：共享依赖本身并不建议，即便所有的团队都是用一个框架，但如果真的有这种需求，可以在微应用中将公共依赖配置成 `external` ,然后在主应用中导入这些依赖

至于为何不建议共享依赖？

官网说qiankun 2.0 版本将提供一种更智能的方式使其自动化。但我看了qiankun github仓库，这个[issues](https://github.com/umijs/qiankun/issues/627)依然处于open状态

大致给出个人觉得可行性的方案：

1. 常用的工具库封装成 `npm` 包，团队管理升降级
    
    缺点：
    
    - 每个微应用都会打包该模块，导致依赖的包冗余，没有真正意义上的复用
    - 当包进行更新发布了，微应用还需要重新构建，调试麻烦低效，虽然可以用npm link来解决
    
2. 使用 `Git Submodule` 管理子模块源码

    同样是依赖npm，与npm不同的是，npm管理的是模块构建产物，git submodule管理的是模块源码，如果不想模块代码暴露出去，可以使用此方式
    
3. `Monorepo`

    单体式仓库，将多个项目放到同一个仓库里面进行管理，统一管理各个模块的构建流程、版本号等。可以避免大量的 `node_modules` 冗余
    
    缺点就是统一构建工具所带来的更高的要求以及仓库体积过大，维护成本高的问题
    
4. `Webpack Externals`

    `externals`中文意思就是外部的，通过在 `externals` 配置项中定义的模块不会存在最终输出的 `bundle` 中
    
    移除掉的模块可以通过 `CDN` 的方式在入口文件中引入，也可以自己预先打包好，再进行引入
    
    缺点就是微应用技术栈多样化的情况，`externals` 并无法支持多版本的情况
5. `Webpack DLL`

    在一个独立的 `webpack` 进行设置 `webpack.dll.config.js`，目的是为了创建一个把所有的第三方库依赖打包到一起的 `bundle` 的 `dll` 文件里面,同时还会生成一个 `manifest.json` 的文件，用于让使用该第三方依赖集合的应用配置的 `DllReferencePlugin` 能映射到相关的依赖上去
    
6. 联邦模块 `Module Federation`
        
   `webpack v5` 新出的一个功能，真正意义上实现跨应用间的模块共享
   
7. 主应用使用 `props` 传递

    主应用通过 `props` 属性传递给子应用，子应用可在入口文件导出的 `bootstrap` 生命周期函数中接收

    缺点：如果微应用单独运行的话，还需要安装一遍
    
**公共样式：**

提取主应用和微应用在主应用的入口文件中引入，子应用可以直接使用

```
// 主应用中定义样式 使用scss定义 并在入口文件中导入
.mr-5 {
    margin-right: 5px;
}

// 微应用页面中使用 同样是使用scss
// f12检查页面元素 会正常解析单位也是 px
<template>
    <div class="mr-5"></div>
</template>

// 微应用页面中使用 使用less
// 会解析为相对单位 1.25rem
```

使用 `scss` 的微应用会正常解析定义的单位,单位同样是 `px`

使用 `less` 的微应用也会正常解析，不过会解析为相对单位 `rem`, 例如 `margin-right: 1.25rem`


## 同样的外部资源主应用加载完成之后，子应用如果也需要加载相同的资源是重新加载？还是用主应用已经加载好的？

可以将一些通用性的库放到主应用加载，第三方库加载后都会抛出一个默认全局对象，比如 `lodash`、`jquery` 这些仓库都会抛出，可以判断 `window` 中是否有这些对象，如果有说明主应用已经加载好；如果没有则重新加载

## 主应用和子应用可以部署在不同的服务器吗？

可以，使用`Nginx`代理进行访问。将主应用服务器上一个特殊路径的请求全部转发到微应用的服务器上。

主应用所在服务器上所有指定开头（例如: /app1)的请求都转发的微应用服务器上

主应用 `Nginx` 配置如下：
```shell
^/app1/ {
    proxy_pass http://www.microapp.com/app1/;
    proxy_set_header Host $host:$server_port;
}

^/api/ {
    proxy_pass http://www.microapp.com/api/;
    proxy_set_header Host $host:$server_port;
}
```

注册微应用时，`entry` 可以为相对路径，`activeRule` 不可以和 `entry` 一样，否则主应用页面刷新就会变成微应用

```javascript
registerMicroApps([
    {
        name: 'app1',
        entry: '/app1/',
        container: '#subapp-container',
        activeRule: '/child/app1'
    }
])
```

`activeRule`需要跟子应用路由的 `base` 一致

## history模式路由刷新404问题如何解决？

举个浏览器解析SPA应用的例子：

SPA应用在输入页面路由时，发送network会发送document请求，返回html文件，浏览器进行解析html文件并加载其中的js和css文件，执行js让`History API`接管页面，重定向到指定路由(例如：http://localhost:8080/login),接着渲染`login`路由所对应页面

当我们再次对页面进行刷新时，服务器同样会尝试根据刷新的URL返回对应的页面，但是服务器上并没有login路由相应的文件。

router（不管是vue-router还是react-router）只是前端来用，并不是给服务器用的

如何防止？

**本地开发：**

```js
// webpack.config.js
module.exports = {
  // ...
  devServer: {
    historyApiFallback: true
  }
}

// webpack.config.js
module.exports = {
  // ...
  devServer: {
    historyApiFallback: {
      rewrites: [
        // to中配置需要根据所配置的publicPath进行配置
        // publicPath配置默认为 /
        // 如果配置为 '/views' 则 to 需要配置为 '/views/index.html'
        // 属性值与HtmlWebpackPlugin所配置的filename属性一致
        { from: /\//, to: '/index.html' }
      ],
      disableDotRule: true // 当在路径中使用点时（在Angular中经常遇到），需要设置此属性为true
    }
  }
}
```

[connect-history-api-fallback文档地址获取更多historyApiFallback配置信息](https://github.com/bripkens/connect-history-api-fallback)

`historyApiFallback (boolean = false object)` : webpack中devServer的配置项，当用到html5中的`History API`的时候,为了确保页面不会相应404的情况，需要设置此属性为true

**Nginx如何调整**

```shell
# 使用root时，实际获取到的静态资源路径是 data/app/index.html
location /app/ {
  root data;
  index index.html index.htm;
  try_files $uri $uri/ /default.html @default;
}


# 使用alias时，实际获取到的静态资源的路径是 data/index.html
location /app/ {
  alias data/;
  index index.html index.htm;
  try_files $uri $uri/ /default.html @default;
}
```

- $uri/，尾部带斜杠，此时表示为一个目录，此时会在目录下查找由index指定的值
- 如果没有找到任何文件，则会对最后一个参数指定的uri进行内部重定向
- attention：最后一个参数是回退URI且必须存在（命名location也可以当做最后一个参数使用），否则会出现内部500错误


**extend：**

**root和alias区别：**

1. root指定路径尾部斜杠可加可不加，alias尾部斜杠是必须要加的
2. root属性值最终会拼接到路径中，alias属性值不会拼接到路径中，会直接在alias属性的值下面去找资源
3. root最终获取的静态页面路径为： 域名 + root属性值 + 匹配规则 + index属性值；alias最终获取的静态页面路径为： 域名 + alias属性值 + index属性值
4. alias是目录别名，root是最上层目录


## 微应用出现接口404问题

以axios为例，baseUrl要设置为绝对路径

例如：`http://localhost:8080/dev-api` 而不是 `/dev-api`

## qiankun下请求子应用静态资源不正常展示，例如图片等资源

**vue3 + vite项目**

解决方式：在`vite.config.ts`文件中的导出的配置中`server`对象增加origin配置

```js
export default defineConfig(({ mode }: ConfigEnv): UserConfig => {
  return {
    server: {
      origin: `http://localhost:${Number(env.VITE_APP_PORT)}`,
      // ...
    }
  }
})
```

**vue2项目**

要确保 `publicPath` 配置正确

确保运行时的 `publicPath` 配置,之前提到的新建 `public-path` 文件，并确保在入口文件中引入，内容为：

```js
if (window.__POWERED_BY_QIANKUN__) {
  __webpack_public_path__ = window.__INJECTED_PUBLIC_PATH_BY_QIANKUN__
}
```

## 微应用打包之后css中的字体文件和图片加载404

`qiankun`将外链样式改成了内联样式，但是字体文件和背景图片的加载路径是相对路径

`css`文件一旦打包完成，就无法通过动态修改 `publicPath` 来修正其中的字体文件和背景图片的路径

- 将所有图片等静态资源上传至 `cdn`, `css` 中直接引用地址
- 借助 `webpack` 的 `url-loader` 将字体文件和图片打包成 `base64` （适用于字体文件和图片体积小的项目)
    
    ```javascript
    // 以vue2项目为例
    // vue.config.js
    
    module.exports = {
        // ...
        chainWebpack: config => {
            config.module
              .rule('images')
              .use('url-loader')
              .tap(options =>
                merge(options, {
                  limit: 5 * 1024
                })
              )
            config.module
              .rule('fonts')
              .test(/\.(png|jpe?g|gif|webp|woff2?|eot|ttf|otf)$/i)
              .use('url-loader')
              .loader('url-loader')
              .tap(options => {
                options = {
                  // limit: 10000,
                  name: '/static/fonts/[name].[ext]'
                }
                return options
              })
              .end()
        }
    }
    ```
- 对于字体和图片比较大的可以使用 `webpack` 的 `file-loader`在打包时给其注入完整路径
    ```javascript
        const publicPath = process.env.NODE_ENV === 'production' ? 'https://qiankun.umijs.org/' : `http://localhost:${port}`;
        module.exports = {
          module: {
            rules: [
              {
                test: /.(png|jpe?g|gif|webp)$/i,
                use: [
                  {
                    loader: 'file-loader',
                    options: {
                      name: 'img/[name].[hash:8].[ext]',
                      publicPath,
                    },
                  },
                ],
              },
              {
                test: /.(woff2?|eot|ttf|otf)$/i,
                use: [
                  {
                    loader: 'file-loader',
                    options: {
                      name: 'fonts/[name].[hash:8].[ext]',
                      publicPath,
                    },
                  },
                ],
              },
            ],
          },
        };
    ```
- 将前两种方案结合起来，小文件转 `base64`,大文件注入路径前缀

## 主应用如何配置404页面？

将404页面配置为一个路由，在主应用的路由钩子函数判断，如果既不是主应用路由也不是微应用路由，就跳转到404页面

```
const childrenPath = ['/app1', '/app2'];
router.beforeEach((to, from, next) => {
  if (to.name) {
    // 有 name 属性，说明是主应用的路由
    return next();
  }
  if (childrenPath.some((item) => to.path.includes(item))) {
    return next();
  }
  next({ name: '404' });
});
```

## 微应用使用 `router.replace` `router.push` 无法正常跳转

有时候需要从当前微应用跳转到另一个微应用中，或者跳转某个页面时会使整个页面重新刷新，状态丢失等

如果都是 `history` 模式路由可使用 `window.history.pushState` 或者 `window.history.replaceState` 方式进行跳转

如果主应用 `hash`模式，子项目使用 `history` 模式，则依然可以借助`window.history.pushState` 或者 `window.history.replaceState` 方式进行跳转

如果都是 `hash` 模式，可以使用 `window.location.hash` 进行跳转

## 微应用切换到主应用的时候，出现主应用未加载css的问题

先复制一下 HTMLHeadElement.prototype.appendChild 和 window.addEventListener ，路由钩子函数 beforeEach 中判断一下，如果当前路由是子项目，并且去的路由是父项目的，则还原这两个对象。

## 主应用下切换微应用出现菜单不显示的情况

微应用入口文件中导出生命周期，将单独渲染的情况分离出来到 `render.js`文件中，在入口文件 `mount` 生命周期中接受 `props` 并使用 `render` 函数渲染微应用

```javascript
// main.js

import { render } from './render'

export async function mount(props) {
    render(props)
}

// render.js
import './permission'
export const render = (props) => {
    // ...
}

```

首次加载微应用时会执行 `permission` 文件的内容，当切换到另一个微应用再切换到当前应用时，却不会进入 `permission` 文件,只会走 `render` 函数里面的逻辑，如果 `redner` 函数中没有加载路由的代码则菜单就不会显示

# 深思熟虑

## 为什么libraryTarget要设置为umd？

这是为了在 `qiankun` 架构下让主应用在执行微应用的js资源时可以通过 `eval`，将 `window` 绑定到一个 `Proxy` 对象上，防止污染全局变量，方便对脚本的window相关操作做劫持处理，达到子应用的脚本隔离

`umd - Universal Module Definition`, 通用模块定义规范，兼容性更高，模块定义的跨平台解决方案，通俗的理解就是可以让代码在  `nodejs` 和浏览器环境中都可以运行

## 什么是运行时的 `publicPath` ?

    [webpack output.publicPath](https://webpack.docschina.org/guides/public-path/#on-the-fly)
    
    
## `npm link`起到什么作用？

开发npm模块的时候,我们会希望边开发边试用，比如本地调试的时候，引入的模块会自动加载本机开发中相应的模块。 `Node` 规定,使用一个模块时，需要将其安装到全局的或项目的 `node_modules` 目录之中。对于开发中的模块，解决方式就是在全局的 `node_modules` 目录之中，生成一个符号链接，指向模块的本地目录

`npm link` 就是起到这个作用，会自动简历符号链接

例如，自己开发了一个模块 `myModule` ,目录为 `src/myModule`,你自己的项目 `myProject` 要用到这个模块，项目目录为 `src/myProject` 。首先，在模块目录 `src/myModule` 下运行 `npm link` 命令。

```
src\myModule> npm link
```

上面的命令会在npm全局模块目录内，生成一个符号链接文件，该文件的名字就是 `package.json`文件中指定的模块名。

这个时候，已经可以全局调用 `myModule` 模块了。但是，如果我们要让这个模块安装在项目内，还需要切换到项目目录，再次运行 `npm link` 命令，并指定模块名

```
src\myProject> npm link myModule
```

上面命令等同于生成了本地模块的符号链接，项目里的 `node_modules/myModule` 就链接到了全局npm模块目录的 `myModule` 模块

`myModule`的任何变化，都可以直接反应在 `myProject`项目之中，**缺点**就是，任何在 `myProject`目录中对 `myModule` 的修改，都会反应到模块的源码中。

如果不再需要该模块，可以在项目目录内使用`npm unlink [package name]` 命令，进行删除符号链接。


# 结语

官方文档只是给出了一个大概雏形，具体的配置还需要自己进行更多的项目实战，在项目中进行总结

博客内容同步更新至 `https://labixiong.github.io/` 网站

    
# 参考文档

- [可能是你见过的最完善的微前端解决方案](https://zhuanlan.zhihu.com/p/78362028)
- [微前端的核心价值](https://zhuanlan.zhihu.com/p/95085796)
- [为什么不是iframe](https://www.yuque.com/kuitos/gky7yw/gesexv)
- [publicPath](https://webpack.docschina.org/guides/public-path/#on-the-fly)
- [micro-frontends](https://micro-frontends.org/)
- [npm link 阮一峰](https://javascript.ruanyifeng.com/nodejs/npm.html#toc18)
- [qiankun中文官网](https://qiankun.umijs.org/zh)
- [微前端模块共享你真的懂了吗](https://juejin.cn/post/6984682096291741704)
- [如何复用公共依赖 目前issues处于open状态](https://github.com/umijs/qiankun/issues/627)
- [webpack官网 Module Federation](https://webpack.js.org/concepts/module-federation/#root)
- [不同的历史模式](https://router.vuejs.org/zh/guide/essentials/history-mode.html)

