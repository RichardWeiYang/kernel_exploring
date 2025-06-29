真实的页分配不仅仅是从free list上取下来就完了，复杂点不在于内存大小的分配，而是对页的分类、统计和管理。

在这里我们先看一个概念 compound page。也就是对一个跨越多个页的“组合页”。
内核对超过一个页面大小的页提供了一个方法PageCompound()来判断一个page是不是组合页的一部分。以下是内核中的注释。

```c
/*
 * Higher-order pages are called "compound pages".  They are structured thusly:
 *
 * The first PAGE_SIZE page is called the "head page" and have PG_head set.
 *
 * The remaining PAGE_SIZE pages are called "tail pages". PageTail() is encoded
 * in bit 0 of page->compound_head. The rest of bits is pointer to head page.
 *
 * The first tail page's ->compound_order holds the order of allocation.
 * This usage means that zero-order pages may not be compound.
 */
```

简单得用一个图来描述这么一个概念。


```
    page[0]               page[1]               page[2]               page[3]
    +----------------+    +----------------+    +----------------+    +----------------+
    |PG_HEAD         |    |_flags_1(order) |    |                |    |                |
    |                |    |                |    |                |    |                |
    |                |    |                |    |                |    |                |
    |                |    |compound_head + |    |compound_head + |    |compound_head + |
    +----------------+    +--------------|-+    +--------------|-+    +--------------|-+
    ^                                    |                     |                     |
    |                                    |                     |                     |
    +------------------------------------+---------------------+---------------------+
```

# 组合页的判断

## 头页面

PageHead

```c
static __always_inline int PageHead(const struct page *page)
{
	PF_POISONED_CHECK(page);
	return test_bit(PG_head, &page->flags) && !page_is_fake_head(page);
}
```

## 尾页面

PageTail

```c
static __always_inline int PageTail(const struct page *page)
{
	return READ_ONCE(page->compound_head) & 1 || page_is_fake_head(page);
}
```

虽然叫尾页，但实际上compound page里，除了头页其余的都是尾页，不仅仅是最后一个。

## 是组合页

PageCompound

```
static __always_inline int PageCompound(const struct page *page)
{
	return test_bit(PG_head, &page->flags) ||
	       READ_ONCE(page->compound_head) & 1;
}
```

这是上面两者的结合，是头页或者是尾页就会返回真。

# 获取头页面

因为重要的信息大多存储在头页面，或者是第二个页面。所以一个组合页找到它的头页面是经常需要的工作。

```
static __always_inline unsigned long _compound_head(const struct page *page)
{
	unsigned long head = READ_ONCE(page->compound_head);

	if (unlikely(head & 1))
		return head - 1;
	return (unsigned long)page_fixed_fake_head(page);
}

#define compound_head(page)	((typeof(page))_compound_head(page))
```

其实就是从page->compound_head中取出来就好了。为什么要这么做，参考set_compound_head()。


# 组合页的分配和释放

设置组合页和清除组合页的方法为

  * prep_compound_page
  * free_tail_pages_check

## 分配流程

当我们从free list上把page拿下来，交出去前，对compound page有额外的处理。这里我们直接从get_page_from_freelist看一下这个流程。

```c
get_page_from_freelist()
    page = rmqueue()                                   // 这里从free list上取下了page
    prep_new_page(page, order, ...)                    // 开始准备page
        post_alloc_hook(page, order, gfp_flags);
        prep_compound_page(page, order);
            __SetPageHead(page);                       // 设置PG_head, page[0]
            prep_compound_tail(page, i);               // 设置compound_head, 所有tail pages
	        p->mapping = TAIL_MAPPING
                set_compound_head(p, head);
                set_page_private(p, 0);
            prep_compound_head(page, order)
                folio = (struct folio*)page;
                folio_set_order(folio, order);
                atomic_set(&folio->_large_mapcount, -1);
                atomic_set(&folio->_entire_mapcount, -1);
                atomic_set(&folio->_nr_pages_mapped, 0);
                atomic_set(&folio->_pincount, 0);
                INIT_LIST_HEAD(&folio->_deferred_list);
```

但是出乎我意料的是，只有在分配页面时使用__GFP_COMP时才会设置compound_head。那么这么说有些“组合页”也不能用PageCompound检测出来了。

## 释放流程

释放过程中对compound page的处理在函数free_pages_prepare().

```c
free_pages_prepare(page, order)
    free_tail_page_prepare(page, page + i)
        page->mapping = NULL;
        clear_compound_head(page)                      // 清除compound_head
    free_page_is_bad(page + i)                         // 检验page是否有问题
    (page + i)->flags &= ~PAGE_FLAGS_CHECK_AT_PREP;
```
