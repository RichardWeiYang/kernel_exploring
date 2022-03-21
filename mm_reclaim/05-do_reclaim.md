纸上得来终觉浅，绝知此事要躬行

看了这么多文章，哪怕是直接看了代码，我以为我懂了，但是真动起手来还是发现有很多不知道，或者是一知半解。

好了，那咱就想办法给系统点压力，让他做个内存回收瞧瞧。

# 触发回收

既然大家都说内存回收在低水线触发，在高水线停止，那么我们想办法让系统可用内存达到低水位就可以了。为了更快达到低水位，我们可以抬高水位，让这个过程更容易达到。

首先我们看一下当前的水线情况。

```
$ free -h
              total        used        free      shared  buff/cache   available
Mem:          3.8Gi       116Mi       3.5Gi       0.0Ki       244Mi       3.7Gi
$ cat /proc/sys/vm/min_free_kbytes
8000
$ grep -E "zone  |min|low |high |managed|pages free" /proc/zoninfo
Node 0, zone      DMA
  pages free     3840
        min      7
        low      10
        high     13
        managed  3840
Node 0, zone    DMA32
  pages free     758226
        min      1505
        low      2257
        high     3009
        managed  758592
Node 0, zone   Normal
  pages free     145291
        min      487
        low      730
        high     973
        managed  243597
Node 0, zone  Movable
  pages free     0
        min      0
        low      0
        high     0
        managed  0
```

约4G的内存，留了8k作为低水位。每个zone可用内存距离低水位还都有些距离。那怎么抬高水位呢？还记得水线计算的公式么？

```
zone的低水线 = (当前zone的内存 * pages_min) / total
```

其中zone的内存和总内存没法改变，所以只能将pages_min提高才能抬高水位。看着当前还有3.5G的空闲内存，那咱就把这最少空闲内存提高到3G吧。

```
echo 3000000 > /proc/sys/vm/min_free_kbytes
$ grep -E "zone  |min|low |high |managed|pages free" /proc/zoneinfo
Node 0, zone      DMA
  pages free     3840
        min      2862
        low      3577
        high     4292
        managed  3840
Node 0, zone    DMA32
  pages free     757824
        min      565534
        low      706917
        high     848300
        managed  758592
Node 0, zone   Normal
  pages free     145221
        min      181602
        low      227002
        high     272402
        managed  243597
Node 0, zone  Movable
  pages free     0
        min      0
        low      0
        high     0
        managed  0
```

怎么样，一下子水位就提高了吧。空闲内存和低水位之间的距离明显缩小。
而且有意思的是Normal Zone的空闲内存已经低于最低水位了，但是这个时候却没有发生回收。你说为啥呢？这是因为在判断节点内存是否需要回收，是按照整个节点纬度判断的。可以看到DMA的zone仍然有750M空闲内存。

另外发现DMA这个Zone的内存实在太小了，从回收的角度很容易达到低水位导致认为整个节点内存已经平衡。所以建议没有特殊需要编译时把ZONE_DMA这个选贤关掉。

```
# memhog 10M
.
# grep pageout /proc/vmstat
pageoutrun 0
```


```
memhog 300M
```

# 谁唤醒了回收

kswapd作为一个内核线程，没事儿的时候他老人家是在那里睡大觉的。只有在需要的时候，才会被唤醒起来干活。
那究竟是谁在什么情况下会集齐龙珠，召唤神龙呢？我们可以看到有一个函数叫wakeup_kswapd，这个是专门用来唤醒kswapd的。
但是这个函数有两个地方被调用，为了确认唤醒kswapd的源头，小编在函数里加了dump_stack()。看看究竟是什么情况会唤醒kswapd老人家。

```
[   27.185421] Call Trace:
[   27.185750]  <TASK>
[   27.186010]  dump_stack_lvl+0x33/0x42
[   27.186453]  wakeup_kswapd.cold.87+0x5/0x35
[   27.186952]  wake_all_kswapds+0x53/0xb0
[   27.187413]  __alloc_pages_slowpath.constprop.136+0xa3f/0xc40
[   27.188120]  ? get_page_from_freelist+0xe3/0xc60
[   27.188673]  __alloc_pages+0x30c/0x320
[   27.189124]  alloc_pages_vma+0x71/0x180
[   27.189610]  __handle_mm_fault+0x315/0xb60
[   27.190103]  handle_mm_fault+0xc0/0x290
[   27.190700]  do_user_addr_fault+0x1d7/0x650
[   27.191375]  exc_page_fault+0x4b/0x110
[   27.191886]  ? asm_exc_page_fault+0x8/0x30
[   27.192447]  asm_exc_page_fault+0x1e/0x30
```

从上面的Trace中可以看到，这是一路从缺页中断过来的。因为内存分配失败，然后唤醒了kswapd。

## 龙珠

刚才说了，要召唤神龙需要集齐龙珠。在召唤kswapd的过程中也需要符合多个条件。其中一条是ALLOC_KSWAPD。

```
if (alloc_flags & ALLOC_KSWAPD)
  wake_all_kswapds(order, gfp_mask, ac);
```

讲真，这个ALLOC_KSWAPD的标志真的让我一顿好找。总结下来，确认分配时需不需要这个标志由以下一个原因判断：

  * alloc_flags从gfp_flags决定，gfp_to_alloc_flags()函数负责这个翻译
  * __GFP_KSWAPD_RECLAIM和ALLOC_KSWAPD的定义是一样的
  * gfp_flags定义中更有多个flag带了__GFP_KSWAPD_RECLAIM，其中基本的两个是__GFP_KSWAPD_RECLAIM和__GFP_RECLAIM
  * 对于匿名页，__alloc_pages的gfp是GFP_HIGHUSER_MOVABLE， 这个定义的展开包含了__GFP_RECLAIM

所以对于普通的用户内存申请都符合这个条件。
