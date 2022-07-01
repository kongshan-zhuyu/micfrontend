## 问题

*Q: 静态资源跨域*
   Access to fetch at ‘http://localhost:3000/’ from origin ‘http://localhost:8000’ has been blocked by CORS policy: No ‘Access-Control-Allow-Origin’ header is present on the requested resource. If an opaque response serves your needs, set the request’s mode to ‘no-cors’ to fetch the resource with CORS disabled.

A: 由于 qiankun 框架 解析微应用使用 import-html-entry 库通过 fetch 请求相关资源，所以需要微应用支持跨域访问；在 webpack devServer 中加入以下代码即可

```js
headers: {
	'Access-Control-Allow-Origin': '*',
}
```

*Q: 主应用增加微应用相关的代理*
   由于子应用在请求时，是使用主应用的服务，所以需要在主应用配置对应的代理，访问子应用的服务。

*Q: 如果样式设置了 `position: fixed;`, 还是根据浏览器窗口位置进行固定位置。*
A: 是的，乾坤对于页面层面的样式，无法进行处理

*Q: Application died in status LOADING_SOURCE_CODE: You need to export the functional lifecycles in xxx entry*
A: 子应用没有暴露生命周期，或主应用没有接收到子应用暴露的生命周期
   这个问题其实在 qiankun 内有着非常详细的排查说明，然而我在对接 qiankun 时，出现的最多的就是这块问题。
   [问题详解](https://qiankun.umijs.org/zh/faq#application-died-in-status-loading_source_code-you-need-to-export-the-functional-lifecycles-in-xxx-entry)

*Q:  webpack5 踩坑*
A: qiankun 使用[webpack5 qiankun 处理](https://github.com/umijs/qiankun/issues/1092)
这个问题，如果 library、libraryTarget 等基础配置都正确，问题主要在 webpack-dev-server 版本问题，升级版本可以解决掉。

对于一些不太能升级 webpack-dev-server 的项目，可以采用下面的办法，算是万金油方案。

在项目入口文件中添加如下代码，先将生命周期函数挂到 window 上

1. 在项目入口文件中添加如下代码，先将生命周期函数挂到 window 上
```js
export async function bootstrap() {
  console.log('bootstrap');
}

export async function mount(props) {
  console.log('mount');
}

export async function unmount(props) {
  console.log('unmount');
}

/**
 * 可选生命周期钩子，仅使用 loadMicroApp 方式加载微应用时生效
 */
export async function update() {
  console.log('update');
}

/**
 *  最理想的方法是升级 webpack-dev-server 到 v4，可以解决问题，
 *  但是，升级到 v4，会产生连锁反应：
 *    1.dev-server 有破坏性更新，原有配置参数需要替换掉
 *    2.升级 dev-server，连带会有其它依赖需要更新，比较难解决
 *
 *  解决办法：
 *    1.先将 qiankun 需要的生命周期函数挂到 window（供别处使用）
 *    2.在 index.html 底部注入 js，将生命周期函数挂到 window['purehtml'] 上
 *      灵感来自：
 *        - https://github.com/umijs/qiankun/issues/1846
 *        - https://qiankun.umijs.org/zh/guide/tutorial#%E9%9D%9E-webpack-%E6%9E%84%E5%BB%BA%E7%9A%84%E5%BE%AE%E5%BA%94%E7%94%A8
 * 
 *  区分运行环境，仅本地调试才需要。
 */
+ if (process.env.NODE_ENV === 'development') {
+   window.qiankunLifecycle = {
+     bootstrap,
+     mount,
+     unmount,
+     update,
+   };
+ }
```
1. index.html 底部添加 script，挂载 qiankun 生命周期函数

借助模板语法 <% %> 控制仅本地调试环境才添加这段 script 代码

window.purehtml 是 qiankun 提供的另一种子应用生命周期挂载方式

```html
<html>
  <head>
  </head>
  <body>
  </body>

  + <% if(process.env.NODE_ENV==='development' ) { %>
  +   <script entry>
  +     if (window.__POWERED_BY_QIANKUN__) {
  +       window.purehtml = window.qiankunLifecycle;
  +     }
  +   </script>
  + <% } %>
</html>
```
注意事项：

- script 标签上必须有 entry 属性（重要）
- script 标签位置必须在 </body> 结束标签后面（重要），否则将出现加载问题.可以为 `html-webpack-plugin` 增加参数 `inject: 'body'`。 

1. 新版本的 html-webpack-plugin 默认会将编译的 js 产物放到 head 标签尾部，并添加 defer 属性，这会导致 qiankun 在应用加载方面产生问题（qiankun 的生命周期 js entry 需要放在页面 js 资源的末尾，即顺序排在最后），尤其当 html 文件底部自己有额外引入 js 文件的情况，一定会出问题。

在 html-webpack-plugin 配置中添加参数 inject: 'body' 搞定。

如果你的编译产物已经是在 </body> 结束标签前，那么无需处理。

1. jsonpFunction 属性需要替换为 chunkLoadingGlobal
[webpack5子应用加载失败](https://github.com/umijs/qiankun/issues/1092)

*Q: 当子应用报错时，控制台的报错信息，没有具体的文件，只能通过不断的试错排查。并不是很友好。*
A: 对于报错信息的收集，并不少很友好

*Q: css 污染问题及加载*
A: qiankun 只能解决子项目之间的样式相互污染，不能解决子项目的样式污染主项目的样式

所以主项目本身的 id/class 需要特殊一点，不能太简单，被子项目匹配到。
