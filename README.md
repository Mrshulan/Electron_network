# **Network Analysis网络分析工具**

## **前言**

写下记录自己开发Network Analysis这么一个小工具的一个过程，想要实现的功能很简单就是 实现一个类似像Chrome浏览器network这个一个小工具(顺带实现了performance性能分析, 正好测试一下之前的博客项目)

这个想法的背景: 当我们测试接口的时候大部分人都是用的postman这个程序来获取 get post delete put 等这样一些restful规则接口，而这似乎没有请求一个document的功能，顶多就是把请求回来的数据也就是简单的将body打印上去

当然也可以使用经典的老大哥抓包神器 Wireshark 或者 Charles 这样的工具，理论上是可行的，但是怎么统计所需要的数据，这可能就是一个难题了

所以我选择了一个集成Chromium & Node.js & Native APIS的Electron来开发一个这样的桌面工具。

![Electron架构](http://qiniu.mrshulan.com/electron%E6%9E%B6%E6%9E%84.png)

## Electron的开发简介

或许你会问我为何没有选择Nw开发，怎么说呢，确实Nw和Electron都可以开发，不过我早对Electron有点了解，而又天天使用github 以及 vscode开发，顺其自然就学了一下Electron(有兴趣的可以去了解一下Electron的背景)

我在Electron的官网找见一个repo[httptoolkit-desktop](https://github.com/httptoolkit/httptoolkit-desktop) 功能有点强大，都可以支持curl的监听，更不用说Chrome、Filefox了，我看了看其代码(看不懂这个组织结构😢)。

Electron的学习过程其实很简单，跟着文档写一些[demo](https://github.com/Mrshulan/Electron_network/tree/master/demo)即可,这里想总结的是里边的进程关系

![Electron进程](http://qiniu.mrshulan.com/electron%E8%BF%9B%E7%A8%8B.png)

这个图里边可以看到一个**主进程(ipcMain)**管理着多个**渲染进程(ipcRenderer)**(每个渲染进程相互独立)

Electron运行时会首先找出(类比一下CRA的入口)`package.json`中的`main`字段标明脚本入口的进程称为**主进程**，在主进程创建web页面来展示用户页面，**一个electron有且只有一个主进程**Electron使用Chromium来展示web页面，每个页面运行在自己的**渲染进程**中。

### 进程通信

- sendMsg 主进程处理渲染进程发来的数据
- sendFeedback 主进程处理渲染进程发来的数据, 并反馈给渲染进程
- sendsync  主进程和渲染进程同步通信

以上的一些方式，都是通过 .send .on 事件监听的模式(类比一下发布订阅模式)，就是一个EventEmitter类的实例，学起来也比较快速，注意一下格式既可

那么渲染进程之间怎么通信呢？可见[demo] news.html开启新窗口

- localstorage(域)
- 窗口id + webContents

渲染进程可以通过electron.remote这个模块可以让渲染进程访问/使用主进程的模块，只需要解构出来，比方说可以开个窗口 `const { remote: {BrowserWindow}} = require('electron');`

通过 `remote` 对象，我们可以不必发送进程间消息来进行通信。但实际上，我们在调用远程对象的方法、函数或者通过远程构造函数创建一个新的对象，实际上都是在发送一个同步的进程间消息

也就是说，`remote` 方法只是不用让我们显式的写发送进程间的消息的方法而已。在上面通过 `remote` 模块创建 `BrowserWindow` 的例子里。我们在渲染进程中创建的 `BrowserWindow` 对象其实并不在我们的渲染进程中，它只是让主进程创建了一个 `BrowserWindow`对象，并返回了这个相对应的远程对象给了渲染进程。

###  一个小问题

我在mac下使用的时候，跑出来的程序默认是不支持 粘贴复制的， 所以需要在menu里边的做一个映射。

## 如何监听到一次网页请求附带的资源请求

Electron提供一个net模块，但是仅仅只支持一次请求，比方说请求https://baidu.com只是拿到一次 header 和 body

这里可能会想到，这已经拿到了body 的document了，我正则匹配之后在递归请求不就可以了？为此我做了一个test，

- 正则如何匹配这样一个庞大的html文件一些外边资源的请求完整性，换句话说，你能不能百分百确保。
- 递归的效率，有没有考虑过。
- 一些请求回来的资源还需要其他的依赖是不是还要考虑，或是要考虑，递归如果死循环了怎么吧，这个边界怎么处理。

这里可以这么做

- Chrome headless模式

  - `chrome --headless --remote-debugging-port=9222 --crash-dumps-dir=/tmp` 监听远程端口(如果不是headless模式则无效果) 也可以在后边直接加需要打开的域名，此时mac会自动打开一个devtools(无GPU界面) 点击小窗口会跳出一个 Websocket监听界面(其原理也是WS)

- chrome-remote-interface & puppeteer

  - 递进思路 headless => chrome-remote-interface (CDP)=> puppeteer => script.js(Node.js)
    - 必须借助headless进行黑窗口监听网络，只需在我们的脚本里边require即可,利用其里边的Client Page Network模块的一些 监听请求函数即可达到全程监听request,`Network.requestWillBeSent、Network.loadingFailed、 Network.loadingFinished、 Network.responseReceived、Network.requestServedFromCache、 `与彼时的Electron net模块相差甚远
    - 如果没有开启headless 就会出错"Error: connect ECONNREFUSED 127.0.0.1:9222"
  - performance分析 借助 Runtime.evaluate 当 `Event.loadEventFired ` mock console.log expression拿到`window.performance.timing.toJSON()` 可做自动化性能测试工具

  这里有个坑

  - client.close & sendFeedbackToRender异步逻辑先后顺序, 如果sendFeedbackToRender处在异步逻辑最后 而 close处任务队列前边直接就没有then, 也轮不到 feedback 反馈。

### 几个问题(除去上面两个)

- 实现一个类型Table 的样式，我采用的是ul li dl dt dd模式,数据一列一列的，而我之前打包的数据的是row的格式，写代码的时候没有思考清楚，直接就是一个map..然后发现不对，显然这样的关系不是这样的，所以需要再次打包成col格式。
- 当传输协议是h2时候，我解构出来的值Content-Length 是0 (分析一波 http2.0是 流和帧的概念 要使用 binary accept-range 解析的好有道理….)， 其实不是这样的(产生的结果不是这个) 而是解构 content-length, 居然headers里边是个小写…..

- 采用Electron 和 React 同时打包的时候 有一个conflict，他们默认同时会涉及build文件 而 CRA npm run build 是提前一步 build 所以 Electron 的默认入口文件 会出现不存在的报错，可在 scripts 后边 使用 -em.main将 主进程文件复制过去，这里操作的时候开始可能会想着，单纯的复制会不会有路径问题，其实多虑了，相对路径，这个相对确实就是平行移动过去的。
- 在优化方面，在 electron-builder 方式下 electron这个是不能支持 -S的, 也就是说他必须是-D安装，原因很简单，Electron打包的时候会默认把dependencies打进去，而这恰好就是一个优化点，能dev的一定要dev。打包后的压缩体积依旧比较大…,Electron 基于 Chromium和Node, 也就是他两必须打包进去..., 本来想看一看electron-react-boilerplate这个优秀的脚手架有没有优化很到位的地方，比方说 webpack大法抽离抽离，然后发现并没有 yarn run build比我的大多了。

## Network Analysis预览

以之前做个博客项目测试一波(那个静态资源的size为何是0可以思考思考🤔)

![运行图片](http://qiniu.mrshulan.com/networkanalysis%E8%BF%90%E8%A1%8C%E5%9B%BE.png)

CSR performance测试结果([线上地址](http://mrshulan.xin/blog/) [项目地址](https://github.com/Mrshulan/Muti_ShareBlog_FE))

![CSR performance](http://qiniu.mrshulan.com/networkblog%E6%B5%8B%E8%AF%95.png)

SSR performance测试结果([线上地址](http://mrshulan.xin/blogssr/) [项目地址](https://github.com/Mrshulan/Muti_ShareBlog_SSR))

![ssr performance](http://qiniu.mrshulan.com/networkblogssr%E6%B5%8B%E8%AF%95.png)

## 其他

如果喜欢 欢迎(孬也)给个star , 来自一个学弟(马上秋招)的乞讨脸.jpg 😂

> 总结我这一年学习的最大感触就是: 知识一定要形成体系
>
> 附上我的代码仓库[**Go-ahead_FE**](https://github.com/Mrshulan/Go-ahead_FE) 里边啥都有真的~