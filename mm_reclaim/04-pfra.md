内存回收的算法，在Gorman的巨著[Understanding the Linux Virtual Memory Manager][1]中有详细的介绍。

[Page Frame Reclaim Algorithm][2]

虽然这部分已经是古董级的材料了，但是作为原理还是很值得研究的。

# LRU

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

其中lists代表的就是大名鼎鼎的lru lists。单枪这个链表一共有五个元素：

	* LRU_INACTIVE_ANON
	* LRU_ACTIVE_ANON
	* LRU_INACTIVE_FILE
	* LRU_ACTIVE_FILE
	* LRU_UNEVICTABLE,

简单来说，回收的过程就是从lru lists上找到合适的page做回收。

## Move between lists

  * 新分配的页：lru_cache_add()，先暂时添加到了pagevec上，再通过__pagevec_lru_add加到lru lists



[1]: https://www.kernel.org/doc/gorman/html/understand/
[2]: https://www.kernel.org/doc/gorman/html/understand/understand013.html
