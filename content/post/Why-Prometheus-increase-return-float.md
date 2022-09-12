---
title: "为什么 Prometheus increase 不返回整数？"
date: 2020-10-04T21:01:59+08:00
draft: false
---
用 Prometheus 做业务监控，需要统计“今日请求量”，很自然想到用 increase 函数。实际效果是它不返回整数，increase难道不是两个数据的差值吗,两个整数相减为什么会有小数点呢？

如果我们查阅官方文档，可以发现delta、increase、rate函数的介绍中都有这么一段类似的话：
    The delta is extrapolated to cover the full time range as specified in the range vector selector, so that it is possible to get a non-integer result even if the sample values are all integers。

这段话翻译过来主要有两层含义：首先，“对整数型metric求差值返回小数”是feature，不是bug；其次，产生小数的原因在于being extrapolated。

## 设计初衷
Prometheus引入extrapolation机制的主要初衷在于解决数据对齐问题。

当用户执行delta、increase或rate等函数时，Prometheus需要对用户指定时间范围内的metric求增值或增长速率。从用户角度来看，metric应当是一条在时间维度上持续延展的曲线，因此对一段时间范围的增值的求解是非常直观的：直接取区间两个端点对应的数值，相减（并除以区间长度）即可得到增值。

然而，从设计角度上来看，Prometheus显然不可能将metric用一条曲线来表示，一是没有足够大的空间来存储线上无数个点，其二是没有足够多的计算资源能够支撑对多个metric的实时收集上报。Prometheus将metric表示为在时间维度上等距的多个离散点，但是用户指定的区间基本上很难和metric区间对齐，即用户区间端点并没有对应的metric数值，因此我们预想的通过端点取值相减是无法实现的。


## 解决思路
prometheus的解决思路很简单, 将metric区间进行等比放大，直至匹配到用户指定的时间范围。其实就是对区间的统计信息做了"线性外插"

举例说明如何做线性外插
如下图：我们每隔 5s 采样一次，问在 [3s, 23s] 的区间内增长了多少？
![](https://gitee.com/ygmyth/blogimage/raw/master/20201004212024.png)
Prometheus 拿到样本的端点 {5s: 10} 与 {20s: 30}，并计算它们的区间为20 - 5 = 15s，期间请求量增长了 30 - 10 = 10 次。因此推算每秒增长了 20/15次，按增长率估算在[3s, 23] 这 20s 期间，增长了 20 * (20/15) = 26.67 次。

线性插值是假设数据线性增长进行推测，而“外插”则表示推测的数据范围数据 [3s, 23s] 在样本点定义域 [5s, 20s] 之外。


## 参考
---

[https://segmentfault.com/a/1190000023717391](https://segmentfault.com/a/1190000023717391)讲解了外插的原理及动机