内存回收的算法，在Gorman的巨著[Understanding the Linux Virtual Memory Manager][1]中有详细的介绍。

[Page Frame Reclaim Algorithm][2]

虽然这部分已经是古董级的材料了，但是作为原理还是很值得研究的。

# LRU定义

回收策略通常被称为Least Recently Used (LRU)。在内核中，对应的数据结构是lruvec。

```
    lruvec
    +-------------------------------+
    |lists[NR_LRU_LISTS]            |
    |    (struct list_head)         |
    |lru_lock                       |
    |    (spinlock_t)               |
    |anon_cost                      |
    |file_cost                      |
    |    (unsigned long)            |
    |nonresident_age                |
    |    (atomic_long_t)            |
    |flags                          |
    |    (unsigned long)            |
    |refaults[ANON_AND_FILE]        |
    |    (unsigned long)            |
    |pgdat                          |
    |    (struct pglist_data*)      |
    +-------------------------------+
```

其中lists代表的就是大名鼎鼎的lru lists。这个上面一共有五个链表：

	* LRU_INACTIVE_ANON
	* LRU_ACTIVE_ANON
	* LRU_INACTIVE_FILE
	* LRU_ACTIVE_FILE
	* LRU_UNEVICTABLE,

简单来说，回收的过程就是从lru lists上找到合适的page做回收。

#把页放到lru上

lru是这样一个数据结构，就好像一个收纳箱。我们把使用的页放在里面，当这个箱子塞满的时候，我们就要清理这个箱子。为了能够更好的清理，我们按照了一定算法在这个箱子里摆放页。这个工作在内核中就是PFRA算法了。

为了更好的理解这个算法，我们可以将这个过程进一步拆解为：

  * 将页存放进箱子和箱子内腾挪的步骤
  * 腾挪操作的算法原理

第一步完全是为了更好理解内核代码做的工程化拆解，也是本小节的主要内容。

## pagevec

半路杀出个程咬金，lruvec的怎么又出来了个pagevec？怎么讲呢，内核为了减少锁竞争，在把页放入lruvec前，先放到percpu的pagevec上。相当于做了一个软cache。

我们先来看看内核中有多少pagevec。


[1]: https://www.kernel.org/doc/gorman/html/understand/
[2]: https://www.kernel.org/doc/gorman/html/understand/understand013.html
