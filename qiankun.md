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

|  | 优点|缺点 |
| -- | -- |--|
|htmL entry|打包配置无需做太多，可以实现预加载|多一层请求，先请求html文件，再用正则匹配到其中的js和css|
|js entry|可以实现按需加载|需要各种打包配置，无法实现预加载|

建议使用 HTML entry ，使用起来和 iframe 一样简单，但是用户体验比 iframe 强很多。qiankun 请求到子项目的 index.html 之后，会先用正则匹配到其中的 js/css 相关标签，然后替换掉，它需要自己加载 js 并运行，然后去掉 html/head/body 等标签，剩下的内容原样插入到子项目的容器中;

<img src="./qiankunimg/img1.png" width="100%">

使用qiankun的好处
- qiankun自带js/css 沙箱功能

js 沙箱的原理是子项目加载之前，对 window 对象做一个快照，子项目卸载时恢复这个快照，如图：


qiankun 的 css 沙箱的原理是重写 HTMLHeadElement.prototype.appendChild 事件，记录子项目运行时新增的 style/link 标签，卸载子项目时移除这些标签。

## qiankun实践

### 改造子应用

``` javascript
// 子应用的入口文件

```