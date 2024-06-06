在伙伴系统的研究中，我们看到page是挂在对应的zone下面的，这个数量是固定的。如果我们每次直接从伙伴系统中获取页，那会导致一个问题。

> 随着cpu个数的增加，对伙伴系统的竞争也会越来越大

linux内核中为了解决这个问题，引入了per_cpu_pageset。

其实理念很简单，就是先从zone这个大仓库里拿一些页出来，放到每个cpu自己的小仓库。用好了，也先放回到这里，等满了再一起还给zone。

那接下来先看看这个结构在zone中是什么样子的。

```
      struct zone
      +------------------------------------------------------------------------------------------------+
      |pageset                                                                                         |
      |   (struct per_cpu_pageset *)                                                                   |
      |   cpu0                          cpu1                                cpuN                       |
      |   +--------------------------+  +--------------------------+  ...   +--------------------------+
      |   |pcp                       |  |pcp                       |        |pcp                       |
      |   |  (struct per_cpu_pages)  |  |  (struct per_cpu_pages)  |        |  (struct per_cpu_pages)  |
      |   |  +-----------------------+  |  +-----------------------+        |  +-----------------------+
      |   |  |count                  |  |  |count                  |        |  |count                  |
      |   |  |high                   |  |  |high                   |        |  |high                   |
      |   |  |batch                  |  |  |batch                  |        |  |batch                  |
      |   |  |                       |  |  |                       |        |  |                       |
      |   |  |lists[MIGRATE_PCPTYPES]|  |  |lists[MIGRATE_PCPTYPES]|        |  |lists[MIGRATE_PCPTYPES]|
      +---+--+-----------------------+--+--+-----------------------+--------+--+-----------------------+
```

可以看到，对于任意一个zone都记录了每个cpu上那个小仓库的信息。具体到每个成员的含义为：

* count: 当前小仓库中有多少页
* high:  如果小仓库里有超过high个数量的页，则还页面到zone
* batch: 如果小仓库没有页面了，则一次从zone中找batch个页面来

这些值还能通过/proc/percpu_pagelist_fraction来动态调节。

那系统中是否能够观察到这些数值呢？也是有办法的。通过/proc/zoneinfo就可以。不过这个文件有点大，大家要仔细去看pageset的字段。

我在这里给大家整理出来一个4个node，8个cpu系统上的zoneinfo中pageset的信息。这样或许有个直观的理解。

```
Node 0, zone      DMA                Node 0, zone    DMA32                  Node 0, zone   Normal                 Node 0, zone  Movable
  pagesets                             pagesets
    cpu: 0                               cpu: 0
              count:  0                            count:  299
              high:   0                            high:   378
              batch:  1                            batch:  63
              vm stats threshold: 8                vm stats threshold: 40
    cpu: 1                               cpu: 1
              count:  0                            count:  86
              high:   0                            high:   378
              batch:  1                            batch:  63
              vm stats threshold: 8                vm stats threshold: 40
    cpu: 2                               cpu: 2
              count:  0                            count:  298
              high:   0                            high:   378
              batch:  1                            batch:  63
              vm stats threshold: 8                vm stats threshold: 40
    cpu: 3                               cpu: 3
              count:  0                            count:  0
              high:   0                            high:   378
              batch:  1                            batch:  63
              vm stats threshold: 8                vm stats threshold: 40
    cpu: 4                               cpu: 4
              count:  0                            count:  0
              high:   0                            high:   378
              batch:  1                            batch:  63
              vm stats threshold: 8                vm stats threshold: 40
    cpu: 5                               cpu: 5
              count:  0                            count:  33
              high:   0                            high:   378
              batch:  1                            batch:  63
              vm stats threshold: 8                vm stats threshold: 40
    cpu: 6                               cpu: 6
              count:  0                            count:  7
              high:   0                            high:   378
              batch:  1                            batch:  63
              vm stats threshold: 8                vm stats threshold: 40
    cpu: 7                               cpu: 7
              count:  0                            count:  0
              high:   0                            high:   378
              batch:  1                            batch:  63
              vm stats threshold: 8                vm stats threshold: 40


Node 1, zone      DMA                Node 1, zone    DMA32                  Node 1, zone   Normal                 Node 1, zone  Movable
                                       pagesets                               pagesets
                                         cpu: 0                                 cpu: 0
                                                   count:  0                              count:  16
                                                   high:   378                            high:   378
                                                   batch:  63                             batch:  63
                                                   vm stats threshold: 32                 vm stats threshold: 32
                                         cpu: 1                                 cpu: 1
                                                   count:  0                              count:  6
                                                   high:   378                            high:   378
                                                   batch:  63                             batch:  63
                                                   vm stats threshold: 32                 vm stats threshold: 32
                                         cpu: 2                                 cpu: 2
                                                   count:  59                             count:  259
                                                   high:   378                            high:   378
                                                   batch:  63                             batch:  63
                                                   vm stats threshold: 32                 vm stats threshold: 32
                                         cpu: 3                                 cpu: 3
                                                   count:  61                             count:  329
                                                   high:   378                            high:   378
                                                   batch:  63                             batch:  63
                                                   vm stats threshold: 32                 vm stats threshold: 32
                                         cpu: 4                                 cpu: 4
                                                   count:  0                              count:  5
                                                   high:   378                            high:   378
                                                   batch:  63                             batch:  63
                                                   vm stats threshold: 32                 vm stats threshold: 32
                                         cpu: 5                                 cpu: 5
                                                   count:  0                              count:  125
                                                   high:   378                            high:   378
                                                   batch:  63                             batch:  63
                                                   vm stats threshold: 32                 vm stats threshold: 32
                                         cpu: 6                                 cpu: 6
                                                   count:  0                              count:  110
                                                   high:   378                            high:   378
                                                   batch:  63                             batch:  63
                                                   vm stats threshold: 32                 vm stats threshold: 32
                                         cpu: 7                                 cpu: 7
                                                   count:  0                              count:  272
                                                   high:   378                            high:   378
                                                   batch:  63                             batch:  63
                                                   vm stats threshold: 32                 vm stats threshold: 32


Node 2, zone      DMA                Node 2, zone    DMA32                  Node 2, zone   Normal                 Node 2, zone  Movable
                                                                              pagesets
                                                                                cpu: 0
                                                                                          count:  165
                                                                                          high:   378
                                                                                          batch:  63
                                                                                          vm stats threshold: 40
                                                                                cpu: 1
                                                                                          count:  175
                                                                                          high:   378
                                                                                          batch:  63
                                                                                          vm stats threshold: 40
                                                                                cpu: 2
                                                                                          count:  291
                                                                                          high:   378
                                                                                          batch:  63
                                                                                          vm stats threshold: 40
                                                                                cpu: 3
                                                                                          count:  137
                                                                                          high:   378
                                                                                          batch:  63
                                                                                          vm stats threshold: 40
                                                                                cpu: 4
                                                                                          count:  122
                                                                                          high:   378
                                                                                          batch:  63
                                                                                          vm stats threshold: 40
                                                                                cpu: 5
                                                                                          count:  362
                                                                                          high:   378
                                                                                          batch:  63
                                                                                          vm stats threshold: 40
                                                                                cpu: 6
                                                                                          count:  322
                                                                                          high:   378
                                                                                          batch:  63
                                                                                          vm stats threshold: 40
                                                                                cpu: 7
                                                                                          count:  198
                                                                                          high:   378
                                                                                          batch:  63
                                                                                          vm stats threshold: 40





Node 3, zone      DMA                Node 3, zone    DMA32                  Node 3, zone   Normal                 Node 3, zone  Movable
                                                                              pagesets
                                                                                cpu: 0
                                                                                          count:  206
                                                                                          high:   378
                                                                                          batch:  63
                                                                                          vm stats threshold: 40
                                                                                cpu: 1
                                                                                          count:  354
                                                                                          high:   378
                                                                                          batch:  63
                                                                                          vm stats threshold: 40
                                                                                cpu: 2
                                                                                          count:  271
                                                                                          high:   378
                                                                                          batch:  63
                                                                                          vm stats threshold: 40
                                                                                cpu: 3
                                                                                          count:  0
                                                                                          high:   378
                                                                                          batch:  63
                                                                                          vm stats threshold: 40
                                                                                cpu: 4
                                                                                          count:  139
                                                                                          high:   378
                                                                                          batch:  63
                                                                                          vm stats threshold: 40
                                                                                cpu: 5
                                                                                          count:  325
                                                                                          high:   378
                                                                                          batch:  63
                                                                                          vm stats threshold: 40
                                                                                cpu: 6
                                                                                          count:  135
                                                                                          high:   378
                                                                                          batch:  63
                                                                                          vm stats threshold: 40
                                                                                cpu: 7
                                                                                          count:  332
                                                                                          high:   378
                                                                                          batch:  63
                                                                                          vm stats threshold: 40
```
