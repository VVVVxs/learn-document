# qiankun 微前端解决方案
微前端是多个小型应用聚合为一的应用，项目独立运行、独立开发、独立部署。
## 解决的问题
- 巨石应用的维护困难。项目过于庞大，人员的更替使得项目代码变得难以维护。
- 技术栈的更替与变迁。项目开发时间长，技术栈老旧，开发效率低性能差。

## 微前端方案调研

### iframe方案

iframe相信大家都很熟悉，使用起来简单方便，提供天然的js/css隔离，也带来数据传输的不方便，本地存储，全局变量和公共插件都没有办法使用。两个项目在不同源的情况下需要使用`postMessage`


``` javascript
//  主应用
import React, { useEffect } from 'react';

function App() {
  useEffect(() => {
    window.addEventListener('message', function (e) {
      console.log('接收到子应用消息',e.data);
    });
  }, [])

  return (
    <div>
      我是主应用
      <iframe src="http://localhost:8080" />
    </div>
  );
}

// 子应用
import React, { useEffect } from 'react';

const App = (props) => {
    useEffect(() => {
        window.parent.postMessage({
            msg: "传送子应用的消息"
        }, 'http://localhost:3000'); // 可以写父应用的域也可以直接写 *
    }, [])
    return (
        <div>
            <div>我是子应用</div>
        </div>
    )
}

```

- 缺点
  - 页面加载问题，每次切换到iframe时， 都需要重新加载。虽然可以用`display:none`来做缓存，但是页面缓存过多容易导致卡顿。
  - iframe必须给一个指定的高度，否则会塌陷。
  - 弹窗和遮罩只能在iframe范围内水平居中，没法在整个页面中水平居中
  - iframe内的div无法全屏
  - 浏览器的前进后退问题，iframe内多次跳转，则会影响浏览器的前进后退功能正常使用。
- 优点
  - 简单方便，接入成本低。
  - 天然隔离了js/css。

### single-spa 微前端方案

spa 单页应用时代，我们的页面只有 index.html 这一个 html 文件，并且这个文件里面只有一个内容标签 <div id="app"></div>，用来充当其他内容的容器，而其他的内容都是通过 js 生成的。

我们只需要拿到子项目的link、script标签，插入到主项目的 HTML 中，就可以在父项目中展现出子项目。

single-spa 是一个微前端框架，基本原理如上，在上述呈现子项目的基础上，还新增了 bootstrap 、 mount 、 unmount 等生命周期。

相比较`iframe`，`single-spa`让父子项目同属一个`document`，这样做既有好处也有坏处。

好处就是数据/文件都可以共享，公共插件共享，子项目加载就更快了。


缺点是带来了 js/css 污染。上手成本大，不能开箱即用，对子项目改造较多。

### qiankun 方案

`qiankun`是基于`single-spa`的基础上做的封装，实现了开箱即用。

`single-spa`原理是将js、css等放在script和link标签中，而`qiankun`既支持这种方法，又支持将子应用整个html都插入到父应中。

|            | 优点                               | 缺点                                                    |
|------------|------------------------------------|---------------------------------------------------------|
| htmL entry | 打包配置无需做太多，可以实现预加载 | 多一层请求，先请求html文件，再用正则匹配到其中的js和css |
| js entry   | 可以实现按需加载                   | 需要各种打包配置，无法实现预加载                        |

建议使用 HTML entry ，使用起来和 iframe 一样简单，但是用户体验比 iframe 强很多。qiankun 请求到子项目的 index.html 之后，会先用正则匹配到其中的 js/css 相关标签，然后替换掉，它需要自己加载 js 并运行，然后去掉 html/head/body 等标签，剩下的内容原样插入到子项目的容器中;

<img src="./qiankunimg/img1.png" width="100%">

使用qiankun的好处
- qiankun自带js/css 沙箱功能

js 沙箱的原理是子项目加载之前，对 window 对象做一个快照，子项目卸载时恢复这个快照，如图：


qiankun 的 css 沙箱的原理是重写 HTMLHeadElement.prototype.appendChild 事件，记录子项目运行时新增的 style/link 标签，卸载子项目时移除这些标签。

## qiankun实践

### 改造子应用

``` javascript
// 子应用的入口文件,增加三个应用的生命周期函数

const render = (props) => {
    ReactDOM.render(<App {...props}/>, document.getElementById("root"));
};
// props中可接受到父应用中传递的参数

// 应用加载之前
export async function bootstrap(props) {
    //
};
// 应用 render 之前触发
export async function mount(props) {
    render({
        name: props.sss,
    })
};
// 应用卸载之后触发
export async function unmount(props) {
    //
};

```

``` javascript
// webpack改造
  module.exports={
    ...
    // 将生命周期函数以umd的方式暴露出去
    output: {
        library: `${name}-[name]`,
        libraryTarget: 'umd',
        jsonpFunction: `webpackJsonp_${name}`,
    },
    // 运行时允许资源跨域，上线之后需要修改下nginx的配置
    devServer: {
        headers: {
            'Access-Control-Allow-Origin': '*',
        }
    },
    ...
  }
```

### 在父应用中引入子应用

- 基于路由配置

当浏览器 url 发生变化时，会自动检查每一个微应用注册的 activeRule 规则，符合规则的应用将会被自动激活。

``` javascript
import { registerMicroApps,start } from 'qiankun';
registerMicroApps([
  {
    name:'app2', // 子应用的唯一标识
    entry:'//localhost:8080', // 微应用的入口
    container:'#app2', // 微应用要挂载的节点
    activeRule:'/app2', // 微应用的激活规则
    loader:(loadingL:boolean)=>void, // loading状态发生变化的回调 可选
    props:object, // 主应用传递给微应用的参数 可选
  }
],{
  // 声明周期函数
  beforeLoad,
  beforeMount,
  afterMount,
  beforeUnmount,
  afterUnmount,
})

// 启动 qiankun。
start();
```

- 手动加载微应用

``` javascript
import React, { useRef } from 'react';
import { loadMicroApp } from 'qiankun';

const App = ()=>{
  const container = useRef(null);
  const loadChildApp = ()=>{
    loadMicroApp({
      name: 'app2',
      entry: '//localhost:8080',
      container: container.current,
    });
  }

  return(
    <div>
      <div ref={container}></div>
      <button onclick={loadChildApp}>加载子应用</button>
    </div>
  )
}
```

### API说明

#### start

- `prefetch`： `boolean` | `'all'` | `string[]` | `function` 可选
  - 配置为 `true` 则会在第一个微应用 mount 完成后开始预加载其他微应用的静态资源
  - 配置为 `'all'` 则主应用 start 后即开始预加载所有微应用静态资源
  - 配置为 `string[]` 则会在第一个微应用 mounted 后开始加载数组内的微应用资源
  - 配置为 `function` 则可完全自定义应用的资源加载时机 (首屏应用及次屏应用)
  - `singular` 可选，是否为单实例场景，单实例指的是同一时间只会渲染一个微应用。默认为 `true`。
- `sandbox`：`boolean` | { `strictStyleIsolation`?: `boolean`, `experimentalStyleIsolation`?: `boolean` } 是否开启沙箱，默认为true

  - 默认情况下是可以确保单实例场景子应用的样式隔离，但是无法确保主应用和子应用，或者多实例场景的子应用样式隔离。当配置为 { `strictStyleIsolation`: `true` } 时表示开启严格的样式隔离模式。这种模式下 qiankun 会为每个微应用的容器包裹上一个 `shadow dom` 节点，从而确保微应用的样式不会对全局造成影响。
  - `experimentalStyleIsolation`：设置`experimentalStyleIsolation:true`时，`qiankun`会改写子应用所添加的样式规则增加一个特殊的选择器来限定范围。
注意: @keyframes, @font-face, @import, @page 将不被支持 (i.e. 不会被改写)

  ``` css
  .app{
    font-size:16px;
  }
  .div[data-qiankun-react16] .app{
    font-size:16px;
  }
  ```

- `singular` - `boolean` | ((`app`: `RegistrableApp`<any>) => `Promise<boolean>`); - 可选，是否为单实例场景，单实例指的是同一时间只会渲染一个微应用。默认为 `true`。

#### setDefaultMountApp

设置主应用启动后默认进入的微应用。

``` javascript

import { setDefaultMountApp } from 'qiankun';
setDefaultMountApp('/app2');

```

#### prefetchApps

手动预加载指定的微应用静态资源。仅手动加载微应用场景需要，基于路由自动激活场景直接配置 prefetch 属性即可。

``` javascript
  import { prefetchApps } from 'qiankun';
  prefetchApps([ { name: 'app1', entry: '//locahost:8080' }, { name: 'app2', entry: '//locahost:8081' } ])
```

#### initGlobalState(state)

定义全局状态，并返回通信方法，建议在主应用使用，微应用通过 props 获取通信方法。

``` javascript
// 初始化
// actions 包含监听，修改，移除监听等方法
const actions: MicroAppStateActions = initGlobalState(state);
```

## 注意事项

### 如何独立运行子应用

根据`window.__POWERED_BY_QIANKUN__`属性判断是否运行在父应用中。如果为`true`就导出父应用所需要的生命周期函数，如果为`false`，就执行`render`函数

### 共享依赖

qiankun并不建议共享依赖，但如果你真的有这个需求，你可以在微应用中将公共依赖配置成 external，然后在主应用中导入这些公共依赖。

### 子应用之间通信

``` javascript
// 主项目初始化
import { initGlobalState } from 'qiankun';
const actions = initGlobalState(state);
// 主项目项目监听和修改
actions.onGlobalStateChange((state, prev) => {
  // state: 变更后的状态; prev 变更前的状态
  console.log(state, prev);
});
actions.setGlobalState(state);

// 子项目监听和修改
export function mount(props) {
  props.onGlobalStateChange((state, prev) => {
    // state: 变更后的状态; prev 变更前的状态
    console.log(state, prev);
  });
  props.setGlobalState(state);
}

```
