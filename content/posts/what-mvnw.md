---
title: "Mvnw是什么鬼"
date: 2020-10-01T17:16:49+08:00
draft: false
---

使用start.spring.io生成项目，会发现里面有mvnw和mvnw.cmd两个文件。两个文件加起来有20多kb。

我代码还没超过20行呢，就整上这样两个文件，做什么用呢？到底是什么鬼？

官方说，它是maven的一个wrapper，在找不到maven的时候，它会自动下载一个；或者，碰到你的项目maven和你环境里的mavne不兼容，它也会自动下载一个。

我们尝试执行一下传统的mvn命令，可以看到它使用mvnw去替换了自己。
```
 ~/codes/ $ mvn -Dmaven.test.skip=true -Pdev package
executing mvnw instead of mvn

Exception in thread "main" java.util.zip.ZipException: error in opening zip file
	at java.util.zip.ZipFile.open(Native Method)
	at java.util.zip.ZipFile.<init>(ZipFile.java:225)
	at java.util.zip.ZipFile.<init>(ZipFile.java:155)
	at java.util.zip.ZipFile.<init>(ZipFile.java:169)
	at org.apache.maven.wrapper.Installer.unzip(Installer.java:169)
	at org.apache.maven.wrapper.Installer.createDist(Installer.java:86)
	at org.apache.maven.wrapper.WrapperExecutor.execute(WrapperExecutor.java:121)
	at org.apache.maven.wrapper.MavenWrapperMain.main(MavenWrapperMain.java:61)
```
但是但是，等了良久，日志也没有向下滚动。等了十几分钟，好不容易有输出了，结果报错。然后接下来每次运行都报错。

聒噪的很，是时候要让它露出真面目了。

使用ps命令，找到了它的启动参数。这才发现，除了mvnw文件，它还偷偷的在项目中放了.mvn目录，好家伙，足足有64kb。
```
# ps -ef| grep mvn
java -classpath ~/codes/.mvn/wrapper/maven-wrapper.jar -Dmaven.home=~/codes -Dmaven.multiModuleProjectDirectory=~/codes  org.apache.maven.wrapper.MavenWrapperMain -Dmaven.test.skip=true -Pdev package
```
这可真是多此一举嘛,真要是贴心，直接塞个apache maven在里面啊。

深处国内，对付这玩意最好的方式，那就是：

**删掉它！删掉它！删掉它！**

即使它的初衷如何好，目标是如何宏大，还是要毫不留情的干掉它。

曾经的我，使用mvnw下载了一下午的jar包，最后茫然地吐槽：公司的maven私服太慢了。

不能背这个锅。

**一个好的项目，不会依赖特定的打包工具。**

这算是maven项目偷懒出的插件，因为一个基础工具，有一个点必须要做到，那就是向后兼容。

搞出这么个工具，连个CDN都舍不得弄，这不是方便开发人员，而是给开发人员添乱。

更要命的是，企业内部都是自己搭建maven私服的，有自己的配置文件和账号。使用这个玩意，还得需要知道maven下载在哪了，找到以后替换它的配置文件。典型的管生不管养啊。

所以，我的处理方式是, 只要看到mvnw和.mvn这些文件，第一时间就毫不留情的干掉它。