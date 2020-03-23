既然谈到了内存的回收再利用，那么就一定有这么一个标准：

> 什么时候开始回收，什么时候结束回收

在内核中这个标准就是 **水线**。

# 水线的含义

水线定义在每个zone上，如下简易示意图所示。

```
    struct zone
    +------------------------------+
    |watermark[NR_WMARK]           |
    |   (unsigned long)            |
    +------------------------------+
```

一共有三条水线：min, low, high。那这三条水线是什么含义呢？用一张图来解释应该比较清楚：

![watermark](/mm_reclaim/vm_watermark.jpg)

从这张图上我们可以看出：

* 当内存容量小于低位水线，则开始启动kswapd回收内存
* 当内存容量小于最小水线，则开启直接回收
* 当内存容量高于高位水线，则kswapd可以休息

概念上还是很清楚的。

# 水线的计算

水线的计算在函数setup_per_zone_wmarks()中完成。不仅启动时会计算，还会在某些参数调整时再次计算。

整个计算过程其实不难，让我用伪代码解释一下：

```
  zone->_watermark[min] = (current zone lowmem * pages_min) / total lowmem

  =>  sum(zone->_watermark[WMARK_MIN]) = pages_min
```

首先系统启动时会计算出一个系统内存保留的最小值： pages_min。这个计算在init_per_zone_wmark_min()完成。
最低水线的计算依赖于这个值。每个zone的最低水线 = (当前zone的lowmem * pages_min) / total lowmem。

这意味着系统上所有zone的最小水线之和 = pages_min 也就是整个系统想要最少预留的内存。

接着就是计算低位和高位水线了：

```
   tmp = (zone_managed_pages(z) * watermark_scale_factor) / 10000

   zone->_watermark[WMARK_LOW]  = min_wmark_pages(zone) + tmp;
   zone->_watermark[WMARK_HIGH] = min_wmark_pages(zone) + tmp * 2;
```

可以看到这三个水线之间的距离是相同的，而这个距离则是当前zone内存容量的一个比例。默认这个比例是0.1%。

好了，是不是挺简单的了。
