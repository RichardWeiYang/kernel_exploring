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

# 开始和结束
