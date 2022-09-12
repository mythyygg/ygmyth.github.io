---
title: "Netty 源码走读 1 - 开始"
date: 2020-10-02T14:27:21+08:00
draft: false
---
本文又是一篇源码分析文章，我是真不太爱写源码分析。因为要花非常多的时间，而且很多地方需要反复组织语言。

本文将介绍 Netty，Java 平台上使用最广泛的 NIO 包，它是对 JDK 中的 NIO 实现的一层封装，让我们能更方便地开发 NIO 程序。其实，Netty 不仅仅是 NIO 吧，但是，基本上大家都冲着 NIO 来的。

个人感觉国内对于 Netty 的吹嘘是有点过了(不是说netty不好)，恨不得把 Netty 吹上天去，使得初学者觉得 Netty 是什么高深的技术一样。

整体读下来的感觉就是,Netty 的源码不很简单，因为它比较多，而且各个类之间的关系错综复杂。我也不知道它的源码好还是不好，真要说好代码，还得 Doug Lea 的并发源码比较漂亮，一行行都是精华，不过它们是不同类型的，也没什么好对比的。

**Netty 源码好就好在它的接口使用比较灵活，往往接口好用的框架，源码都不会太简单。**

源码分析，需要先掌握 NIO 的基础知识。本文只介绍 TCP 相关的内容，不讨论Netty 对于其他协议的支持。

Netty 源码一直在更新，各版本之间有些差异，我是按照 2019-04 的最新版本 4.1.34.Final 来进行介绍的。

## 准备
学习源码，一开始肯定是准备环境。

我喜欢用 maven，也喜欢 Spring Boot，所以我一般先到 https://start.spring.io/ 准备一个最简单的脚手架。

10 秒搞定脚手架，然后就是导入到 Intellij 中，如果用新版本的 Spring Boot，可能还需要等待下载依赖。

Netty 分为好些模块，有 netty-handler、netty-buffer、netty-transport、netty-common 等等，也有一个 netty-all，它包含了所有的模块。

既然我们是源码分析，那么自然是用一个最简单的。netty-all 不是最好的选择，netty-example 才是：

```xml
<dependency>
   <groupId>io.netty</groupId>
   <artifactId>netty-example</artifactId>
   <version>4.1.25.Final</version>
</dependency>
```
它不仅可以解决我们的依赖，而且 example 里面的示例非常适合我们学习使用。

## Echo 例子
Netty 作为 NIO 的库，自然既可以作为服务端接受请求，也可以作为客户端发起请求。使用 Netty 开发客户端或服务端都是非常简单的，Netty 做了很好的封装，我们通常只要开发一个或多个 handler 用来处理我们的自定义逻辑就可以了。

下面，我们来看一个经常会见到的例子，它叫 Echo，也就是回声，客户端传过去什么值，服务端原样返回什么值。

打开 netty-example 的源码，把 echo 包下面的代码复制出来玩一玩。
![](https://gitee.com/ygmyth/blogimage/raw/master/20201002143408.png)

左边是服务端代码，右边是客户端代码。

上面的代码基本就是模板代码，每次使用都是这一个套路，唯一需要我们开发的部分是 handler(…) 和 childHandler(…) 方法中指定的各个 handler，如 EchoServerHandler 和 EchoClientHandler，当然 Netty 源码也给我们提供了很多的 handler，比如上面的 LoggingHandler，它就是 Netty 源码中为我们提供的，需要的时候直接拿过来用就好了。

我们先来看一下上述代码中涉及到的一些内容：

- ServerBootstrap 类用于创建服务端实例，Bootstrap 用于创建客户端实例。

- 两个 EventLoopGroup：bossGroup 和 workerGroup，它们涉及的是 Netty 的线程模型，可以看到服务端有两个 group，而客户端只有一个，它们就是 Netty 中的线程池。

- Netty 中的 Channel，没有直接使用 Java 原生的 ServerSocketChannel 和 SocketChannel，而是包装了 NioServerSocketChannel 和 NioSocketChannel 与之对应。

    *当然，也有对其他协议的支持，如支持 UDP 协议的 NioDatagramChannel，本文只关心 TCP 相关的。*

- 左边 handler(…) 方法指定了一个 handler（LoggingHandler），这个 handler 是给服务端收到新的请求的时候处理用的。右边 handler(...) 方法指定了客户端处理请求过程中需要使用的 handlers。

    如果你想在 EchoServer 中也指定多个 handler，也可以像右边的 EchoClient 一样使用 ChannelInitializer

    左边 childHandler(…) 指定了 childHandler，这边的 handlers 是给新创建的连接用的，我们知道服务端 ServerSocketChannel 在 accept 一个连接以后，需要创建 SocketChannel 的实例，childHandler(…) 中设置的 handler 就是用于处理新创建的 SocketChannel 的，而不是用来处理 ServerSocketChannel 实例的。

- pipeline：handler 可以指定多个（需要上面的 ChannelInitializer 类辅助），它们会组成了一个 pipeline，它们其实就类似拦截器的概念，现在只要记住一点，每个 NioSocketChannel 或 NioServerSocketChannel 实例内部都会有一个 pipeline 实例。pipeline 中还涉及到 handler 的执行顺序。

- ChannelFuture：这个涉及到 Netty 中的异步编程，和 JDK 中的 Future 接口类似。

本文主要是一个入门介绍, 列出我觉得比较难理解或比较重要的部分。上面的源码中没有展示消息发送和消息接收的处理，此部分在后面几篇文章中再做介绍。