
内存回收有那么点儿复杂，也涉及到了系统中很多地方。为了对这些做一个梳理，我整理了回收的大图。这样可以对全局有个了解。

下图中左边部分就是所谓的直接回收(direct reclaim)， 右边部分就是kswapd了。

![vmscan](/mm_reclaim/vmscan.png)
