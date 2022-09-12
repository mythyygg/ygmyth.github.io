---
title: "SecureRandom 采坑记录"
date: 2020-10-02T16:30:18+08:00
draft: false
---
## 背景
我们的项目工程里经常在每个函数需要用到 Random 的地方定义一下 Random 变量（如下）
```java
public void doSomethingCommon() {
  Random rand = new Random();
  ...
}
```
在用 sonar 进行检查时，会发现会有如下告警
```
Creating a new Random object each time a random value is needed is inefficient and may produce numbers which are not random depending on the JDK. For better efficiency and randomness, create a single Random, then store, and reuse it.
```

简单地说就是在每个函数都创建一个 Random 效率太低了，而且由于 JDK 版本的不同，可能 Random 产生的随机数不够随机。为了提升性能和随机性，建议定义一个 Random 单例来统一产生随机数, Sonar 建议使用 SecureRandom.getInstanceStrong() 来初始化,如下
```java
private Random rand = SecureRandom.getInstanceStrong();
```
于是我就将其改成 sonar 建议的形式来生成随机数

## 问题初现
结果问题来了，在我们业务的接口上,需要随机返回一批用户头像，这个接口返回很多502状态码的报警.从监控上看就是接口执行阻塞.
## 定位问题
问题是上线2小时之后出现,猜测是上线功能有关,先回滚.

**复现问题**：首先使用了相同的请求参数在测试进行了测试，但令人不解的是，问题无法复现。随后又测试了线上机器，可以稳定的复现问题。这时一脸黑人问号。

**打日志**：为了定位问题，在代码的关键位置插入日志，经过多次的发布定位到问题，执行到 SecureRandom.getInstanceStrong() 方法后就阻塞了。

## 原因分析
根本原因是SecureRandom 这个jre的工具类的问题.
具体内容：[JDK-6521844 : SecureRandom hangs on Linux Systems](https://bugs.java.com/bugdatabase/view_bug.do?bug_id=6521844)

    SecureRandom.getInstanceStrong() 方法在 linux 环境下使用 /dev/random 生成种子。但是 /dev/random 是一个阻塞数字生成器，如果它没有足够的随机数据提供，它就一直等，这迫使 JVM 等待。键盘和鼠标输入以及磁盘活动可以产生所需的随机性或熵。但在一个缺乏这样的活动服务器，可能会出现问题，当系统的熵池中数量不足时，就会阻塞当前线程。

那么 Linux 中随机数是如何产生的呢 PRNG(Pseudo-Random Number Generator)

Linux 内核采用熵来描述数据的随机性，熵（entropy）是描述系统混乱无序程度的物理量，一个系统的熵越大则说明该系统的有序性越差，即不确定性越大。内核维护了一个熵池用来收集来自设备驱动程序和其它来源的环境噪音。理论上，熵池中的数据是完全随机的，可以实现产生真随机数序列。为跟踪熵池中数据的随机性，内核在将数据加入池的时候将估算数据的随机性，这个过程称作熵估算。熵估算值描述池中包含的随机数位数，其值越大表示池中数据的随机性越好。内核中随机数发生器 PRNG 为一个字符设备 random,代码实现在 drivers/char/random.c，该设备实现了一系列接口函数用于获取系统环境的噪声数据，并加入熵池。系统环境的噪声数据包括设备两次中断间的间隔，输入设备的操作时间间隔，连续磁盘操作的时间间隔等。

random 设备了提供了 2 个字符设备供用户态进程使用——/dev/random 和/dev/urandom：

/dev/random 适用于对随机数质量要求比较高的请求，在熵池中数据不足时， 读取 dev/random 设备时会返回小于熵池噪声总数的随机字节。/dev/random 可生成高随机性的公钥或一次性密码本。若熵池空了，对/dev/random 的读操作将会被阻塞，直到收集到了足够的环境噪声为止。这样的设计使得/dev/random 是真正的随机数发生器，提供了最大可能的随机数据熵。
/dev/urandom，非阻塞的随机数发生器，它会重复使用熵池中的数据以产生伪随机数据。这表示对/dev/urandom 的读取操作不会产生阻塞，但其输出的熵可能小于/dev/random 的。它可以作为生成较低强度密码的伪随机数生成器，对大多数应用来说，随机性是可以接受的。


## 解决方案:
有了以上的的解释，我们就知道解决方案了，使用 /dev/urandom 这种非阻塞的方式来产生随机数即可解决问题,在 Java 中我们改成如下写法即可解决问题
```
SecureRandom random = new SecureRandom();
```