
# 回收的大致逻辑

下面整理了一下回收中个人认为比较重要的调用关系。

  * 左边从kswapd()开始的是间接回收
  * 中间从node_reclaim()开始的是直接回收（省略了其他的部分）
  * 最右边是memcg对内存的限制（猜测）

```
kswapd() // one thread for each node
    |
    v
balace_pgdat(pgdat, )                node_reclaim()
    |                                    |
    v                                    v
kswapd_shrink_node(pgdat, )          __node_reclaim(pgdat, )
    \                                      /
         \                            /
              \                  /
                   \        /
                        v
                 shrink_node(pgdat, )                           mem_cgroup_soft_reclaim()
                     |                                                |
                     v                                                v
                 shrink_node_memcgs(pgdat, )                    mem_cgroup_shrink_node(memcg, )
                     memcg = mem_cgroup_iter()                      lruvec = mem_cgroup_lruvec(memcg, pgdat)
                     lruvec = mem_cgroup_lruvec(memcg, pgdat)

                             \                                      /
                                  \                            /
                                       \                  /
                                            \        /
                                                 v
                                         shrink_lruvec(lruvec, )
                                                 |
                                                 v
                                           shrink_list(lruvec, )
                                                 ^
                                            /          \
                                       /                    \
                                  /                              \
                       shrink_active_list()               shrink_inactive_list()
                                                                 |
                                                                 |
                                                                 v
                                                          shrink_folio_list(&folio_list, )
                                                                 |
                                                                 |
                                                                 v
                                                              pageout()
```

但是不管怎么样的路径下来，最后都是走到了shrink_lruvec(lruvec,)，那自然他就是我们重点观照对象。

并且从名字上看出，我们回收内存时管理的最小对象是lruvec。

# priority

priority是一个整型，语义上表示重要程度：值越小，优先级越高。

数值上表示，每次回收过程中扫描(total_pages >> priority)数量的页。所以数值越小，优先级越高，扫描的页面越多。

# swappiness

swappiness是一个可以调节的参数，/proc/sys/vm/swappiness，表示在文件页和匿名页之间更倾向于回收哪种。

这个的作用在get_scan_count()->calculate_pressure_balance()中体现。

我们可以分别将swappiness为0和MAX_SWAPPINESS代入，得到ap是0,或者fp是0。表明

  * swappiness = 0: 只回收文件页
  * swappiness = MAX_SWAP: 只回收匿名页

# shrink_list

在shrink_lruvec()中，通过get_scan_count()和其他逻辑计算出了各链表上需要扫表的数量后，就通过shrink_list()来做扫描。

  * shrink_active_list: 扫描活跃链表，从链表尾部开始扫描，将合适的页面移动到inactive list
  * shrink_inactive_list: 扫描不活跃链表，合适的时候通过shrink_folio_list()回收


