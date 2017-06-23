# Bigview
> 现代web站点变得更加动态和内容化，交互性也越来越强，传统的页面处理方式已越来越不能够满足用户对性能的追求。

## 传统的交互模型
首先我们先来看看传统的交互模型：

* 浏览器发送HTTP请求给服务器
* 服务器解析来自客户端的请求，从存储层（缓存或数据库等）获取到数据，生成HTML页面，给客户端发送HTTP响应。
* 客户端解析响应，开始构建DOM Tree,然后开始下载CSS和JavaScript。
* CSS下载完毕，解析CSS并继续生成DOM Tree。
* JavaScript下载完毕，被浏览器解析执行。

传统的页面交互模型按照一定的顺序来执行的，每个过程都是不可重叠的，即每个过程不能再同一时间被执行。当服务器获取数据并生成页面的时候，客户端被闲置，等待服务器生成数据；当客户端接收到服务前端返回的页面并开始下载资源，解析页面的时候，服务器有在等待来自客户端的下一次请求。空闲的时间造成资源的浪费。

基于这个现象，FaceBook的人就开始想了，如果客户端能够在服务器生成页面的时候同时能够进行资源的下载和页面的解析，在页面进行资源下载和解析的过程服务器端也能够继续生成页面，那么整体的性能将会得到很大的提升。于是，Bigpipe就诞生了。

## Bigpipe的交互模式
为了让一个页面能够同时被客户端和服务前端处理，首先我们需要将一个完整的页面划分成若干个小块，这些小块bigpipe将其称作pagelets。然后通过Bigpipe技术，让页面以pagelet的形式在服务前端生成并分块发送给客户端（write,write....）。
Bigpipe让页面的生成步骤拆成以下几个步骤：

* 服务前端接收来自客户端的HTTP请求
* 从存储层获取数据
* 生成HTML，响应给客户端
* 浏览器解析内容，开始下载CSS，浏览器生成DOM Tree,生成CSS Rule Tree,构造Render Tree,绘制和显示元素。
* 下载并执行JavaScript。

前面三步在服务前端完成，后面三步在客户端完成。每一个pagelet都要执行以上所有步骤，但Bigpipe可让这些pagelet并行的去完成这些步骤。

### 那么让给我们来看看Bigpipe究竟是怎么工作的？
首先客户端给服务前端发送一个HTTP请求，服务前端首先生成一段不闭合的HTML片段，包含<head>和不闭合的<body>标签，在head里面包含了处理后续接收到的pagelet的BigPipe库，然后响应给客户端。

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

`<script type="text/javascript">
big_pipe.onPageletArrive({id: “pagelet_composer”, content=<HTML>, css=[..], js=[..], …})
</script>`

一个pagelet包含了id、HTML片段、依赖的CSS和JavaScript资源。

在客户端，当接收到一个pagelet（此时服务前端还在继续生成其他的pagelet）时，马上执行onPageletArrive方法，Bigpipe库会根据返回的pagelet信息立即开始下载CSS资源，下载完成之后会子啊页面上线上pagelet片段。


重复上面的步骤，直到pagelet全部处理完毕，整个流程结束，pagelet处理的过程服务前端和客户端一直保持着同步工作状态。
