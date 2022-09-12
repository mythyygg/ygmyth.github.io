---
title: "[总结] 性能监控与故障处理工具"
date: 2020-10-04T16:00:18+08:00
draft: false
tags: [ "标签1", "标签2", "标签3" ]
categories: [ "分类1", "分类2" ]
---
JVM 的理论不想写了，本文介绍一些可以实操的 JVM 性能监控与分析工具。

JDK 自带的几个常用工具如下：

|名称	| 主要作用 |
| --- | --- |
|jps |	JVM Process Status Tool, 显示指定系统内所有的 HotSpot 虚拟机进程 |
|jstat |	JVM Statistics Monitoring Tool, 用于收集 HotSpot 虚拟机各方面的运行数据 |
|jinfo |	Configuration Info for Java, 显示虚拟机配置信息 |
|jmap |	Memory Map for Java, 生成虚拟机的内存转储快照（heapdump 文件）|
|jhat |	JVM Heap Analysis Tool, 用于分析 heapdump 文件（它会建立一个 HTTP/HTML 服务器，让用户可以在浏览器上查看分析结果）|
|jstack |	Stack Trace for Java, 显示虚拟机的线程快照 |

这里只是少部分，其他更多命令可以参考官方文档：https://docs.oracle.com/javase/8/docs/technotes/tools/unix/toc.html

## jps: 虚拟机进程状况工具

命令格式

```
jps [ options ] [ hostid ]
```

- jps
```
$ jps
15236 Jps
14966 Example1
```
- jps -l
```
$ jps -l
15249 sun.tools.jps.Jps
14966 com.jaxer.jvm.egs.Example1
```
- jps -m
```
$ jps -m
15264 Jps -m
14966 Example1
```
- jps -v
```
$ jps -v
14966 Example1 -Dvisualvm.id=44321340563858 -Xmx50m -Xms50m -XX:+PrintGCDetails -javaagent:/Applications/IntelliJ IDEA.app/Contents/lib/idea_rt.jar=61849:/Applications/IntelliJ IDEA.app/Contents/bin -Dfile.encoding=UTF-8
15278 Jps -Dapplication.home=/Library/Java/JavaVirtualMachines/jdk1.8.0_191.jdk/Contents/Home -Xms8m
```
- jps -q
```
$ jps -q
9938
14966
15334
```
## jstat: 虚拟机统计信息监视工具
命令格式
```
jstat [option vmid [interval[s|ms] [count]] ]
```

- 示例 1：监控堆内存信息
```
$ jstat -gc 14966
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT
2048.0 2048.0  0.0    0.0   12800.0   9345.8   34304.0    26638.8   5248.0 4971.3 640.0  554.9       2    0.032   2      0.049    0.082
```
如图所示：
![](https://gitee.com/ygmyth/blogimage/raw/master/20201004161602.png)

参数主要是新生代、老年代的内存空间占用情况以及 GC 的次数和时间，说明如下：
```
S0: Survivor space 0 utilization as a percentage of the space's current capacity.

S1: Survivor space 1 utilization as a percentage of the space's current capacity.

E: Eden space utilization as a percentage of the space's current capacity.

O: Old space utilization as a percentage of the space's current capacity.

M: Metaspace utilization as a percentage of the space's current capacity.

CCS: Compressed class space utilization as a percentage.

YGC: Number of young generation GC events.

YGCT: Young generation garbage collection time.

FGC: Number of full GC events.

FGCT: Full garbage collection time.

GCT: Total garbage collection time.
```
官方文档：https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html#BEHHGFAE


示例 2：每隔 1000 毫秒打印堆内存信息，打印 10 次
```
$ jstat -gc 14966 1000 10
```
如图所示：
![](https://gitee.com/ygmyth/blogimage/raw/master/20201004161702.png)


示例 3：查看类加载/卸载信息
```
$ jstat -class 14966
Loaded  Bytes  Unloaded  Bytes     Time
   829  1604.4        0     0.0       0.37
```
## jinfo: Java 配置信息工具

查看 JVM 启动参数
```
$ jinfo -flags 26472
VM Flags:
-XX:CICompilerCount=3 -XX:InitialHeapSize=52428800 -XX:MaxHeapSize=52428800 -XX:MaxNewSize=17301504 -XX:MinHeapDeltaBytes=524288 -XX:NewSize=17301504 -XX:OldSize=35127296 -XX:+PrintGCDetails -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseFastUnorderedTimeStamps -XX:+UseParallelGC
```
值得注意的是，JDK 8 使用该命令时会抛出如下异常：
```
$ jinfo -flags 26284
Attaching to process ID 26284, please wait...
Error attaching to process: sun.jvm.hotspot.debugger.DebuggerException: Can't attach symbolicator to the process
sun.jvm.hotspot.debugger.DebuggerException: sun.jvm.hotspot.debugger.DebuggerException: Can't attach symbolicator to the process
 at sun.jvm.hotspot.debugger.bsd.BsdDebuggerLocal$BsdDebuggerLocalWorkerThread.execute(BsdDebuggerLocal.java:169)
 at sun.jvm.hotspot.debugger.bsd.BsdDebuggerLocal.attach(BsdDebuggerLocal.java:287)
 at sun.jvm.hotspot.HotSpotAgent.attachDebugger(HotSpotAgent.java:671)
 at sun.jvm.hotspot.HotSpotAgent.setupDebuggerDarwin(HotSpotAgent.java:659)
 at sun.jvm.hotspot.HotSpotAgent.setupDebugger(HotSpotAgent.java:341)
 at sun.jvm.hotspot.HotSpotAgent.go(HotSpotAgent.java:304)
 at sun.jvm.hotspot.HotSpotAgent.attach(HotSpotAgent.java:140)
 at sun.jvm.hotspot.tools.Tool.start(Tool.java:185)
 at sun.jvm.hotspot.tools.Tool.execute(Tool.java:118)
 at sun.jvm.hotspot.tools.JInfo.main(JInfo.java:138)
 at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
 at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
 at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
 at java.lang.reflect.Method.invoke(Method.java:498)
 at sun.tools.jinfo.JInfo.runTool(JInfo.java:108)
 at sun.tools.jinfo.JInfo.main(JInfo.java:76)
Caused by: sun.jvm.hotspot.debugger.DebuggerException: Can't attach symbolicator to the process
 at sun.jvm.hotspot.debugger.bsd.BsdDebuggerLocal.attach0(Native Method)
 at sun.jvm.hotspot.debugger.bsd.BsdDebuggerLocal.access$100(BsdDebuggerLocal.java:65)
 at sun.jvm.hotspot.debugger.bsd.BsdDebuggerLocal$1AttachTask.doit(BsdDebuggerLocal.java:278)
 at sun.jvm.hotspot.debugger.bsd.BsdDebuggerLocal$BsdDebuggerLocalWorkerThread.run(BsdDebuggerLocal.java:144)
 ```
查资料说是 JVM 的 bug（链接：https://bugs.java.com/bugdatabase/view_bug.do?bug_id=8160376），在 JDK 9 b129 修复了。

因此，这里测试该命令使用了 JDK 11，版本信息如下：
```
$ java -version
java version "11.0.5" 2019-10-15 LTS
Java(TM) SE Runtime Environment 18.9 (build 11.0.5+10-LTS)
Java HotSpot(TM) 64-Bit Server VM 18.9 (build 11.0.5+10-LTS, mixed mode)
```
## jstack: Java 堆栈跟踪工具
jstack (Stack Trace for Java) 命令用于生成虚拟机当前时刻的线程快照（一般称为 threaddump 或 javacore 文件）。
线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合，生成堆栈快照的主要目的是定位线程出现长时间停顿的原因，如线程死锁、死循环、请求外部资源导致的长时间等待等都是导致线程长时间停顿的常见原因。
```
$ jstack -l 26472 | more
```
如图所示：
![](https://gitee.com/ygmyth/blogimage/raw/master/20201004162026.png)

## jmap: Java 内存映像工具
查看对象信息
可以使用 jmap 查看内存中的对象数量及内存空间占用：
```
jmap [option] <pid>
```
例如：
![](https://gitee.com/ygmyth/blogimage/raw/master/20201004162136.png)

导出堆转储快照
```
$ jmap -dump:live,format=b,file=/Users/jaxer/Desktop/dump.hprof 26472
Dumping heap to /Users/jaxer/Desktop/dump.hprof ...
Heap dump file created
```
可以在对应路径下看到堆转储快照文件 dump.hprof。导出来之后，就可以用其它工具分析快照文件了。

## jhat: 堆转储快照分析工具
分析 jmap 生成的快照文件，命令如下：
```
$ jhat dump.hprof
Reading from dump.hprof...
Dump file created Sun May 03 17:09:07 CST 2020
Snapshot read, resolving...
Resolving 42293 objects...
Chasing references, expect 8 dots........
Eliminating duplicate references........
Snapshot resolved.
Started HTTP server on port 7000
Server is ready.
```
Server 启动后，在浏览器打开 http://localhost:7000/，可以看到如下信息：
![](https://gitee.com/ygmyth/blogimage/raw/master/20201004162209.png)
实际工作中，一般不会直接使用 jhat 命令来分析 dump 文件，主要原因：

一般不会在部署应用程序的服务器上直接分析 dump 文件（分析工作一般比较耗时，而且消耗硬件资源，在其他机器上进行时则没必要受到命令行工具的限制）；
jhat 分析功能相对简陋，VisualVM 等更工具功能强大。
## jvisualvm & VisualVM: 堆转储快照分析工具
jvisualvm 也是 JDK 自带的命令，虽然后面独立发展了。这两种方式都可以使用。

VisualVM 链接：https://visualvm.github.io/

使用 VisualVM 分析上面 jmap 导出的堆栈转储文件，导入后如下：

- 概览
![](https://gitee.com/ygmyth/blogimage/raw/master/20201004162228.png)

- 对象信息
![](https://gitee.com/ygmyth/blogimage/raw/master/20201004162251.png)


- 线程信息
![](https://gitee.com/ygmyth/blogimage/raw/master/20201004162301.png)

这个工具使用起来比 jhat 舒服多了。

## jconsole: JVM 性能监控
使用 jconsole 命令会启动一个用户界面，如下：
![](https://gitee.com/ygmyth/blogimage/raw/master/20201004162458.png)


可以选择本地或者远程的 JVM 进程进行连接。

本人测试连接时报错，描述及解决方案详见：https://blog.csdn.net/u010551118/article/details/105907403

连接成功之后如下：


- 概览

![](https://gitee.com/ygmyth/blogimage/raw/master/20201004162516.png)

- 死锁检测
![](https://gitee.com/ygmyth/blogimage/raw/master/20201004162545.png)


