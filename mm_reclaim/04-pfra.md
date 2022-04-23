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

```
    lru_pvecs                                  lru_rotate
    +-------------------------------+          +-------------------------------+
    |lock                           |          |lock                           |
    |    (local_lock_t)             |          |    (local_lock_t)             |
    |lru_add                        |          |pvec                           |
    |lru_deactivate_file            |          |    (struct pagevec)           |
    |lru_deactivate                 |          +-------------------------------+
    |lru_lazyfree                   |
    |activate_page                  |
    |    (struct pagevec)           |
    |    +--------------------------+          mlock_pvec(struct pagevec)
    |    |pages[PAGEVEC_SIZE]       |          +-------------------------------+
    |    |    (struct page*)        |          |                               |
    |    |nr                        |          +-------------------------------+
    |    |    (unsigned char)       |
    |    |percpu_pvec_drained       |
    |    |    (bool)                |
    |    |                          |
    +----+--------------------------+
```

考虑到内核中还有别的子系统使用pagevec，这里只列出和lru相关的。所以这么数来，一共有七个相关的pagevec。而对于每一个pagevec，内核中都有对应的函数处理。咱们先把相关的函数展示出来。

```
                                folio_rotate_reclaimable
	                              lru_rotate.pvec
                                      |

                                      |  folio_activate                    deactivate_page
	                                       lru_pvecs.activate_page           lru_pvecs.lru_deactivate
                                      |       /                              /
                                             /                         /
  folio_add_lru                       |  deactivate_file_folio             mark_page_lazyfree
  lru_pvecs.lru_add                      lru_pvecs.lru_deactivate_file     lru_pvecs.lru_lazyfree
           \                          |   /          /                       /
                    \                 |  /     /                 /
                            v         v v     v      v
                          pagevec_add_and_need_flush
                                   /     \
                           /                     \
                  __pagevec_lru_add           pagevec_lru_move_fn



     mlock_page_drain     mlock_folio      mlock_new_page   munlock_page
             \                  \            /                 /
                      \          \          /         /
                               \  \        / /
                                mlock_pagevec
                                   /     \
                           /                     \
                __mlock_page   __mlock_new_page  __munlock_page
```

本来我想把这两个合一块的，社区没同意。也好，那就分开看看。

先解释一下上面的图：

  * mlock_pvec 比较独立。添加到mlock_pvec后，由mlock_pagevec加到lru上
  * 其余的pagevec都通过pagevec_add_and_need_flush检查后，做相应的操作
  * folio_add_lru/mlock_new_page 是两个加入到pagevec的入口函数


[1]: https://www.kernel.org/doc/gorman/html/understand/
[2]: https://www.kernel.org/doc/gorman/html/understand/understand013.html
