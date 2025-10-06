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

# lruvec在哪里

lruvec是可以理解成一个系统中已经分配除去的页面的集散地，目的是为了后续释放而存在的。要找到这个lruvec在哪里，我们就要看folio_lruvec()这个函数。

毕竟内核演化这么多年了，这个lruvec可能出现在两个不同的位置。PS：当然不是同时出现。

  * pgdat->__lruvec: 没有memcg，或者有memcg但是disable的情况下
  * memcg的lruvec:   有memcg且enable的情况下

所以在操作lruvec时，会先找到folio对应的memcg，然后去操作。

# 把页放到lru上之前

lru是这样一个数据结构，就好像一个收纳箱。我们把使用的页放在里面，当这个箱子塞满的时候，我们就要清理这个箱子。为了能够更好的清理，我们按照了一定算法在这个箱子里摆放页。这个工作在内核中就是PFRA(Page Frame Reclaim Algorithm)算法了。

为了更好的理解这个算法，我们可以将这个过程进一步拆解为：

  * 将页存放进箱子和箱子内腾挪的步骤
  * 腾挪操作的算法原理

第一步完全是为了更好理解内核代码做的工程化拆解，也是本小节的主要内容。但是再放到lru之前，内核为了减少竞争，增加了一个存放的缓存空间。

## cpu_fbatches

说实话，下面的pagevec我已经全忘记了。现在,2025.10.06，重看这部分代码，pagevec已经消失了。如果我没有猜错的话，替代pagevec的，就是cpu_fbatches。

首先，这也是一个percpu的变量，可以也是一个加入到lru之前的缓冲区。


```
    cpu_fbatches
    +-------------------------------+
    |lock                           |
    |    (local_lock_t)             |
    |lru_add                        |
    |lru_deactivate_file            |
    |lru_deactivate                 |
    |lru_lazyfree                   |
    |lru_activate                   |
    |    (struct folio_batch)       |
    |    +--------------------------+
    |    |i                         |
    |    |nr                        |
    |    |    (unsigned char)       |
    |    |pages[PAGEVEC_SIZE]       |
    |    |    (struct page*)        |
    |    |percpu_pvec_drained       |
    |    |    (bool)                |
    |    +--------------------------+
    |lock_irq                       |
    |    (local_lock_t)             |
    |lru_move_tail                  |
    |    (struct folio_batch)       |
    +-------------------------------+
```

这么看，其实和原来的lru_pvecs很像，不过是用folio_batch替换了之前的pagevec。然后再一看，folio_batch和pagevec也是很像的。所以这个其实就是从page到folio的一个变化。

对应的commit也可以看出这个变化。

```
commit 10331795fb7991a39ebd0330fdb074cbd81fef48
Author: Matthew Wilcox (Oracle) <willy@infradead.org>
Date:   Mon Dec 6 15:24:51 2021 -0500

    pagevec: Add folio_batch
    
    The folio_batch is the same as the pagevec, except that it is typed
    to contain folios and not pages.
```

除了数据结构的变化，操作上也发生了变化。我猜现在操作上主要都几种到了函数folio_batch_add_and_move()。

```
static void __folio_batch_add_and_move(struct folio_batch __percpu *fbatch,
		struct folio *folio, move_fn_t move_fn, bool disable_irq)
{
	unsigned long flags;

	folio_get(folio);

	if (disable_irq)
		local_lock_irqsave(&cpu_fbatches.lock_irq, flags);
	else
		local_lock(&cpu_fbatches.lock);

	if (!folio_batch_add(this_cpu_ptr(fbatch), folio) ||
			!folio_may_be_lru_cached(folio) || lru_cache_disabled())
		folio_batch_move_lru(this_cpu_ptr(fbatch), move_fn);

	if (disable_irq)
		local_unlock_irqrestore(&cpu_fbatches.lock_irq, flags);
	else
		local_unlock(&cpu_fbatches.lock);
}

#define folio_batch_add_and_move(folio, op)		\
	__folio_batch_add_and_move(			\
		&cpu_fbatches.op,			\
		folio,					\
		op,					\
		offsetof(struct cpu_fbatches, op) >=	\
		offsetof(struct cpu_fbatches, lock_irq)	\
	)
```

不得不说，内核开发者这些c语言大师对代码的精妙处理。这种代码都写得出来。

第一步是将folio添加到对应的folio_batch中，起到了缓存的作用。如果对应的folio_batch满了，才会使用folio_batch_move_lru()，并通过对应的move_fn对folio进行处理。:

```
static void folio_batch_move_lru(struct folio_batch *fbatch, move_fn_t move_fn)
{
	int i;
	struct lruvec *lruvec = NULL;
	unsigned long flags = 0;

	for (i = 0; i < folio_batch_count(fbatch); i++) {
		struct folio *folio = fbatch->folios[i];

		/* block memcg migration while the folio moves between lru */
		if (move_fn != lru_add && !folio_test_clear_lru(folio))
			continue;

		folio_lruvec_relock_irqsave(folio, &lruvec, &flags);
		move_fn(lruvec, folio);

		folio_set_lru(folio);
	}

	if (lruvec)
		unlock_page_lruvec_irqrestore(lruvec, flags);
	folios_put(fbatch);
}
```

从cpu_fbatches的定义可以看出，这里的move_fn有6种可能性:

  * lru_add
  * lru_deactivate_file
  * lru_deactivate
  * lru_lazyfree
  * lru_activate
  * lru_move_tail

而这些就是将folio放到对应lruvec上链表的具体操作了。

## pagevec -- 老版本

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

# 将folio放到lru

在上面的分析中，我们看到folio_batch_move_lru()函数接收的move_fn可以有下面六中情况:

  * lru_add
  * lru_deactivate_file
  * lru_deactivate
  * lru_lazyfree
  * lru_activate
  * lru_move_tail

这些才是真正操作lruvec的地方。

比如lru_add将folio添加到lruvec->lists[lru]; lru_deactivate将folio从active上取下，放到deactivate的链表上。

[1]: https://www.kernel.org/doc/gorman/html/understand/
[2]: https://www.kernel.org/doc/gorman/html/understand/understand013.html
