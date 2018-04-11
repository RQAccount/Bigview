# Bigview
> 现代web站点变得更加动态和内容化，交互性也越来越强，传统的页面处理方式已越来越不能够满足用户对性能的追求。

## 传统的交互模型
首先我们先来看看传统的交互模型：

- 浏览器发送HTTP请求给服务器
- 服务器解析来自客户端的请求，从存储层（缓存或数据库等）获取到数据，生成HTML页面，给客户端发送HTTP响应。
- 客户端解析响应，开始构建DOM Tree,然后开始下载CSS和JavaScript。
- CSS下载完毕，解析CSS并继续生成DOM Tree。
- JavaScript下载完毕，被浏览器解析执行。

传统的页面交互模型按照一定的顺序来执行的，每个过程都是不可重叠的，即每个过程不能再同一时间被执行。当服务器获取数据并生成页面的时候，客户端被闲置，等待服务器生成数据；当客户端接收到服务前端返回的页面并开始下载资源，解析页面的时候，服务器有在等待来自客户端的下一次请求。空闲的时间造成资源的浪费。

基于这个现象，FaceBook的人就开始想了，如果客户端能够在服务器生成页面的时候同时能够进行资源的下载和页面的解析，在页面进行资源下载和解析的过程服务器端也能够继续生成页面，那么整体的性能将会得到很大的提升。于是，Bigpipe就诞生了。

## 什么是Bigpipe？
- 存在很久的一种分块加载技术
- Facebook首创
- 首屏快速加载的的异步加载页面方案
- 前端性能优化的一个方向
- 适合比较大型的，需要大量服务器运算的站点
- 有效减少HTTP请求
- 兼容多浏览器

与传统Ajax比较
- 减少HTTP请求数：多个模块更新合成一个请求
- 请求数减少：多个chunk合成一个请求
- 减少开发成本：前端无需多写JavaScript代码
- 降低管理成本：模块更新由后端程序控制
- URL优雅降级：页面链接使用真实地址
- 代码一致性：页面加载不劢态刷新模块代码相同

解决的问题
- 下载阻塞
- 服务器和浏览器资源浪费问题

### Bigpipe的交互模式
为了让一个页面能够同时被客户端和服务前端处理，首先我们需要将一个完整的页面划分成若干个小块，这些小块bigpipe将其称作pagelets。然后通过Bigpipe技术，让页面以pagelet的形式在服务前端生成并分块发送给客户端。
Bigpipe让页面的生成步骤拆成以下几个步骤：

- 服务前端接收来自客户端的HTTP请求
- 从存储层获取数据
- 生成HTML，响应给客户端
- 浏览器解析内容，开始下载CSS，浏览器生成DOM Tree,生成CSS Rule Tree,构造Render Tree,绘制和显示元素。
- 下载并执行JavaScript。

前面三步在服务前端完成，后面三步在客户端完成。每一个pagelet都要执行以上所有步骤，但Bigpipe可让这些pagelet并行的去完成这些步骤。

### Bigpipe究竟是怎么工作的？
首先客户端给服务前端发送一个HTTP请求，服务前端首先生成一段不闭合的HTML片段，包含<head>和不闭合的<body>标签，在head里面包含了处理后续接收到的pagelet的BigPipe库，然后响应给客户端。

```
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="HandheldFriendly" content="true">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0, user-scalable=0">
  <meta name="apple-mobile-web-app-capable" content="yes">
  <title>BigPipe Test</title>
  <link rel="stylesheet" href="bigpipe.styl">
  <script src="bigpipe.js"></script>
</head>
<body>
```

这个时候服务前端会继续去生成pagelets,而客户端接收到响应已经开始下载CSS和其他资源。当服务前端生成好一个pagelet的时候，就会立即flush到客户端，pagelet的格式如下：

```
<script type="text/javascript">
big_pipe.onPageletArrive({id: “pagelet_composer”, content=<HTML>, css=[..], js=[..], …})
</script>
```

一个pagelet包含了id、HTML片段、依赖的CSS和JavaScript资源。

在客户端，当接收到一个pagelet（此时服务前端还在继续生成其他的pagelet）时，马上执行onPageletArrive方法，Bigpipe库会根据返回的pagelet信息立即开始下载CSS资源，下载完成之后会子啊页面上线上pagelet片段。


重复上面的步骤，直到pagelet全部处理完毕，整个流程结束，pagelet处理的过程服务前端和客户端一直保持着同步工作状态。
### Bigpipe在性能上是究竟达到了怎么的飞跃？
假设客户端向服务前端发送一个页面的HTTP请求，该页面包括三个模块（pagelet）,那么传统方式和BigePpipe从服务器接收到HTTP请求到浏览器完全完成渲染该页面的不一样呢？

首先我们先假定服务器生成一个模块、网络传输一个模块和浏览器解析渲染一个模块均需1s。下面我们看下图：
![...](/images/responseModeCpmpare.jpg)
传统模式三个模块完全生成需要3s,第4s开始进入网络传输，又需经过3s网络传输才能结束，第7s浏览器才接收到第一个模块（即第7s进行首屏渲染），第10s才完成整个页面的渲染，整个页面从服务器到浏览器需要9s的时间。
而Bigpipe在第一个pagelet（A_S）生成后就直接进入传输层，也就是说第2s的时候就进行网络传输，比传统模式快了2s,Bigpipe前后端并行，所以A_s生成完成后进入网络层的同时，服务器会继续生成其他pagelet,完成后也直接进入网络层，无需再等待其他pagelet生成完毕。同理网络层接收到服务器生成的pagelet进行传输，一个pagelet传输完毕，浏览器端就接收数据进行解析渲染，无需等到所以pagelet传输完毕。如此我们可看到，第3s时浏览器就接收到第一个模块（即第3s进行首屏渲染），比传统模式快了4s；第6s完成整个页面的渲染，整个页面从服务器到浏览器需要5s的时间，比传统模式快了4s。

经过上面的对比，我们可以看到Bigpipe在性能上做了加到的改进，效果达到了质的飞跃。
]

### 怎么实现服务前端准备就绪一个pagelet模块,客户端就能够收到响应进行渲染显示一个pagelet呢？—— 关键的技术点
首先服务前端准备好一个pagelet后，就会通过res.write()响应给客户端。
而现在几乎所有的主流浏览器都支持HTTP 1.1版本，而HTTP 1.1版本的Tranfer-Encoding  chunked  特性支持一个模块一个模块的接收并渲染。当服务前端所有pagelet生成完毕后，向客户端响应res.end()就可以结束一个请求的响应。

HTTP 1.1 引入分块传输编码
> 注：HTTP分块传输编码允许服务器为动态生成的内容维持HTTP持久链接。

HTTP分块传输编码格式
> Transfer-Encoding: chunked 如果一个HTTP消息（请求消息或应答消息）的Transfer-Encoding消息头的值为chunked，那么，消息体由数量未定的块组成，并以最后一个大小为0的块为结束。

Nodejs自动开启 chunked encoding
> 除非通过sendHeader()设置Content-Length头。

## Bigview是什么？
Bigview是使用Node.js、Promise（bluebird）和 Bigpipe技术构建的框架。

### 组件
- bigview(Node.js)视图基类 
- biglet(Node.js)模块基类
- bigeview.js（前端引用）。

### 功能点
- 支持静态布局和动态布局
- 支持5种bigpipe渲染模式
  - parallel.js   并行模式， 先写布局，并行请求，但在获得所有请求的结果后再渲染
  - pipeline.js  (默认) 管线模式：即并行模式， 先写布局，并行请求，并即时渲染
  - reduce.js    顺序模式： 先写布局，按照pagelet加入顺序，依次执行，写入
  - reducerender.js 先写布局，然后顺序执行，在获得所有请求的结果后再渲染
  - render.js 一次渲染模式：即普通模式，不写入布局，所有pagelet执行完成，一次写入到浏览器。支持搜索引擎，用来支持那些不支持JS的客户端。
- 支持子pagelet，无限级嵌套
- 支持根据条件渲染模板，延时输出布局
- bigview支持错误模块显示，仅限于布局之前


下面我们再重新看看服务器端和浏览器端是怎么工作的。

![...](/images/responseProcess.jpg)
浏览器端发出请求，服务器端响应 res ,  首先向浏览器端返回布局，浏览器端接收后进行首屏渲染，与此同时服务器端还在准备模块1、模块2....模块n，一个模块准备就绪就给浏览器端吐。直到 所有模块准备好并向浏览器端写入了，服务器端调res.end()浏览器接到该信息后domComplete，接着进入domReady 状态。

### 生命周期

#### bigview的生命周期

- before
    - then(this.beforeRenderLayout.bind(this))
    - then(this.renderLayout.bind(this))
    - then(this.afterRenderLayout.bind(this))
    - then(this.beforeRenderPagelets.bind(this))
    - then(this.renderPagelets.bind(this))
    - then(this.afterRenderPagelets.bind(this)
- end

#### bigview的生命周期精简

- before
- renderLayout
- renderPagelets
- end

#### biglet的生命周期

- before
    - then(self.fetch.bind(self))
    - then(self.parse.bind(self))
    - then(self.render.bind(self))
- end

### Node.js bigpipe实现

#### 使用内置的http模块

``` js
'use strict'

var http = require('http')

const sleep = ms => new Promise(r => setTimeout(r, ms))

var app = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/html', 'charset': 'utf-8' })
  
  res.write('loading...<br>')
  
  return sleep(2000).then(function() {
    res.write(`timer: 2000ms<br>`)
    return sleep(5000)
  })
  .then(function() {
    res.write(`timer: 5000ms<br>`)
  }).then(function() {
    res.end()
  })
})

app.listen(3000)
```

#### Express写法

``` js
'use strict'

const sleep = ms => new Promise(r => setTimeout(r, ms))

var express = require('express')
var app = express()

app.get('/', function (req, res) {
  res.type('html');   
  res.write('loading...<br>')
  
  return sleep(2000).then(function() {
    res.write(`timer: 2000ms<br>`)
    return sleep(5000)
  })
  .then(function() {
    res.write(`timer: 5000ms<br>`)
  }).then(function() {
    res.end()
  })
})

app.listen(3000)
``` 
#### Koa 1.x

``` js
var koa = require('koa')
var app = koa()
var co = require('co')
const Readable = require('stream').Readable

const sleep = ms => new Promise(r => setTimeout(r, ms))

app.use(function* () {
  const view = new Readable()
  view._read = () => { }

  this.body = view
  this.type = 'html'
  this.status = 200

  view.push('loading...<br>')

  co(function* () {
      yield sleep(2000)
      view.push(`timer: 2000ms<br>`)
      yield sleep(5000)
      view.push(`timer: 5000ms<br>`)

      /** 结束传送 */
      view.push(null)
  }).catch(e => { })
})

app.listen(9092)
```

#### Koa 2.x

``` js
const Koa = require('koa')
const app = new Koa()

const sleep = ms => new Promise(r => setTimeout(r, ms))

app.use(require('koa-bigpipe'))

// response
app.use(ctx => {
  // ctx.body = 'Hello Koa'
  ctx.write('loading...<br>')
  
  return sleep(2000).then(function(){
    ctx.write(`timer: 2000ms<br>`)
    return sleep(5000)
  }).then(function(){
    ctx.write(`timer: 5000ms<br>`)
  }).then(function(){
    ctx.end()
  })
})

app.listen(3000)
```

#### 关键字

    res.write('xxxx');
    res.write('xxxx');
    res.write('xxxx');
    res.end('xxxx');

#### 为什么不用res.send?

    因为res.send包括了res.write()和res.end()

#### 为什么是按照顺序加载的,怎么能并发加载呢?
    这就需要用到promise了, 自行领悟

### Bigview & Biglet 关系

![](/images/bigview&biglet.jpg)
