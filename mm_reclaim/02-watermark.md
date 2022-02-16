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
                        = (current zone lowmem / total lowmem) * pages_min
                        = (proportionate to the zone's size) * pages_min
  then we can get
  =>  sum(zone->_watermark[WMARK_MIN]) = pages_min
```

首先系统启动时会计算出整个系统内存保留的最小值： pages_min。这个计算在init_per_zone_wmark_min()完成。
最低水线的计算依赖于这个值。每个zone的最低水线 = (当前zone的lowmem * pages_min) / total lowmem。

这意味着系统上所有zone的最小水线之和 = pages_min 也就是整个系统想要最少预留的内存。

PS: 这个pages_min的当前值可以从/proc/sys/vm/min_free_kbytes查看，不过需要转换一下。

接着就是计算低位和高位水线了：

```
   tmp = (zone_managed_pages(z) * watermark_scale_factor) / 10000

   zone->_watermark[WMARK_LOW]  = min_wmark_pages(zone) + tmp;
   zone->_watermark[WMARK_HIGH] = min_wmark_pages(zone) + tmp * 2;
```

可以看到这三个水线之间的距离是相同的，而这个距离则是当前zone内存容量的一个比例。默认这个比例是0.1%。

PS: 这里的watermark_scale_factor也可以从/proc/sys/vm/watermark_scale_factor查看。

好了，是不是挺简单的了。

## 水位查看

了解了计算过程，我们也可以来看看当前系统中真实设置的值。这个信息可以在/proc/zoneinfo中查到。

```
Node 0, zone   Normal
  pages free     2761
        boost    1459
        min      1946
        low      2189
        high     2432
        spanned  262144
        present  262144
        managed  243597
```

其中managed的页有243597，那么默认情况0.1%就是243个page。从上面的数据看，正好符合这个公式。

```
low  = min + 243
high = low + 243
```

然后我们看一下这个水位对一个zone究竟是什么比例。

```
min      1946      = 7.6M
low      2189      = 8.5M
high     2432      = 9.5M
managed  243597    = 951M
```

可以看到，对一个1G的zone来讲，高水位也就10M，而每个水位之间间隔1M。
这么看，要唤醒kswapd，那是内存已经非常紧张的时候了。

# lowmem_reserve

除了水线，还有一个概念控制着什么时候进行内存回收 -- lowmem_reserve。准确的来说应该是这是水线的组成部分。

在判断水线的函数__zone_watermark_ok()中，我们可以看到这么一个判断：

```
    /*
     * Check watermarks for an order-0 allocation request. If these
     * are not met, then a high-order request also cannot go ahead
     * even if a suitable page happened to be free.
     */
    if (free_pages <= min + z->lowmem_reserve[classzone_idx])
      return false;
```

也就是空闲页需要大于 水线(min) + lowmem_reserve[]才能算是水线达标。那么我们就来看看这个lowmem_reserve是怎么计算的吧。

setup_per_zone_lowmem_reserve()是设置lowmem_reserve[]的函数，让我们用一张图来展示一下这个概念：

```
        Zone[0]       Zone[1]      Zone[2]      Zone[3]
        +-------------+-------------+------------+------------+
  ls[3] |mp[3] + mp[2]|mp[3] + mp[2]|mp[3]       |     0      |
        |+ mp[1] /    |   /         |   /        |            |
        |ra[0]        |ra[1]        |ra[2]       |            |
        +-------------+-------------+------------+------------+
  ls[2] |mp[2] + mp[1]|mp[2]        |     0      |   N/A      |
        |   /         |   /         |            |            |
        |ra[0]        |ra[1]        |            |            |
        +-------------+-------------+------------+------------+
  ls[1] |mp[1]        |     0       |   N/A      |   N/A      |
        |   /         |             |            |            |
        |ra[0]        |             |            |            |
        +-------------+-------------+------------+------------+
  ls[0] |     0       |   N/A       |   N/A      |   N/A      |
        |             |             |            |            |
        |             |             |            |            |
        +-------------+-------------+------------+------------+
```

其中采用的缩写标示为：

* mp[i]: managed pages of zone[i]
* ra[i]: sysctl_lowmem_reserve_ratio[i]
* ls[i]: lowmem_reserve[i]

这么看来 zone.lowmem_reserve[i]的含义是**如果要在zone上分配 zone[i]的内存，而此时zone上的内存小于zone.lowmem_reserve[i] + watermark，那么这个分配将出发回收动作。**.
