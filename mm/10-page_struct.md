页结构体(struct page)可以说是我见过的最为繁杂的结构体定义了，至今我只能说是略懂。然而这个结构体定义又是那么的重要，因为物理意义上的每个页面都对应一个结构体。可以说这个结构体是整个系统上所有内存的守护者。

每个细分成员的作用就不在本节展开了，这里主要是作为一个全局参考，从整体上对页结构体有个了解。具体的作用将在对应的章节中描述。

# 庞然大物

下面这个图已经是很老的状态了，自从有了folio后简化了不少。我还是放在这里，以后可以有个对照。看看folio简化的效果。

```
    struct page (include/linux/mm_types.h)

                         page
               +--------------------------------------------------------------+
               |flags                                                         |    page-flags-layout.h
               |   (unsigned long)                                            |    page-flags.h
   --+--       +==============================================================+
     |         |..............................................................|
     |         |page cache and anonymous pages                                |
     |         |    +---------------------------------------------------------+
     |         |    |lru                                                      |
     |         |    |    (struct list_head)                                   |
               |    |mapping                                                  |
               |    |    (struct address_space*)                              |
               |    |index                                                    |
               |    |    (pgoff_t)                                            |
  5 words      |    |private                                                  |
   union       |    |    (unsigned long)                                      |
               |    +---------------------------------------------------------+
               |..............................................................|
    has        |slab, slob, slub                                              |
               |    +---------------------------------------------------------+
  7 usage      |    |.........................................................|
               |    |                            .+---------------------------|
               |    |slab_list                   .|next                       |
               |    |    (struct list_head)      .|   (struct page*)          |
               |    |                            .|pages                      |
               |    |                            .|pobjects                   |
               |    |                            .|   (int)                   |
               |    |                            .+---------------------------|
               |    |.........................................................|
               |    +---------------------------------------------------------+
               |    |slab_cache                                               |
               |    |    (struct kmem_cache*)                                 |
               |    |freelist                                                 |
               |    |    (void*)                                              |
               |    +---------------------------------------------------------+
               |    |.........................................................|
               |    |s_mem    .counters          .+---------------------------|
               |    | (void*) . (unsigned long)  .|inuse                      |
               |    |         .                  .|objects                    |
               |    |         .                  .|frozen                     |
               |    |         .                  .|    (unsigned)             |
               |    |         .                  .+---------------------------|
               |    |.........................................................|
               |    +---------------------------------------------------------+
               |..............................................................|
               |Tail pages of compound page                                   |
               |    +---------------------------------------------------------+
               |    |compound_head                                            |
               |    |    (unsigned long)                                      |
               |    |compound_dtor                                            |
               |    |compound_order                                           |
               |    |    (unsigned char)                                      |
               |    |compound_mapcount                                        |
               |    |    (atomic_t)                                           |
               |    +---------------------------------------------------------+
               |..............................................................|
               |Second tail page of compound page                             |
               |    +---------------------------------------------------------+
               |    |_compound_pad_1                                          |
               |    |_compound_pad_2                                          |
               |    |    (unsigned long)                                      |
               |    |deferred_list                                            |
               |    |    (struct list_head)                                   |
               |    +---------------------------------------------------------+
               |..............................................................|
               |Page table pages                                              |
               |    +---------------------------------------------------------+
               |    |_pt_pad_1                                                |
               |    |    (unsigned long)                                      |
               |    |pmd_huge_pte                                             |
               |    |    (pgtable_t)                                          |
               |    |_pt_pad_2                                                |
               |    |    (unsigned long)                                      |
               |    |.........................................................|
               |    |pt_mm                         .pt_frag_refcount          |
               |    |    (struct mm_struct*)       .    (atomic_t)            |
               |    |.........................................................|
               |    |ptl                                                      |
               |    |    (spinlock_t/spinlock_t *)                            |
               |    +---------------------------------------------------------+
               |..............................................................|
               |ZONE_DEVICE pages                                             |
               |    +---------------------------------------------------------+
               |    |pgmap                                                    |
               |    |    (struct dev_pagemap*)                                |
               |    |hmm_data                                                 |
               |    |_zd_pad_1                                                |
     |         |    |    (unsigned long)                                      |
     |         |    +---------------------------------------------------------+
     |         |..............................................................|
     |         |rcu_head                                                      |
     |         |    (struct rcu_head)                                         |
     |         |..............................................................|
   --+--       +==============================================================+
     |         |..............................................................|
               |            .                 .                .              |
   4 bytes     |_mapcount   .page_type        .active          .units         |
    union      |  (atomic_t).   (unsigned int).  (unsigned int).   (int)      |
               |            .                 .                .              |
     |         |..............................................................|
   --+--       +==============================================================+
               |_refcount                                                     |
               |     (atomic_t)                                               |
               |mem_cgroup                                                    |
               |     (struct mem_cgroup)                                      |
               |virtual                                                       |
               |     (void *)                                                 |
               |_last_cpupid                                                  |
               |     (int)                                                    |
               +--------------------------------------------------------------+
```

# page->flags

在页结构体众多成员中，flags字段在任何用途下都标示了对应页的属性。除此之外内核中还对这个字段做了非常巧（变）妙（态）的布局。

内核中有两个头文件和flags的定义有着密切的联系

    * include/linux/page-flags-layout.h
    * include/linux/page-flags.h

前者定义了该字段的布局，后者则定义了该字段具体的意义和相关操作的宏定义。

## 布局

以下对字段布局的描述摘自page-flags-layout.h

```
/*
 * page->flags layout:
 *
 * There are five possibilities for how page->flags get laid out.  The first
 * pair is for the normal case without sparsemem. The second pair is for
 * sparsemem when there is plenty of space for node and section information.
 * The last is when there is insufficient space in page->flags and a separate
 * lookup is necessary.
 *
 * No sparsemem or sparsemem vmemmap: |       NODE     | ZONE |             ... | FLAGS |
 *      " plus space for last_cpupid: |       NODE     | ZONE | LAST_CPUPID ... | FLAGS |
 * classic sparse with space for node:| SECTION | NODE | ZONE |             ... | FLAGS |
 *      " plus space for last_cpupid: | SECTION | NODE | ZONE | LAST_CPUPID ... | FLAGS |
 * classic sparse no space for node:  | SECTION |     ZONE    | ... | FLAGS |
 */
```

其中FLAGS部分的定义就在include/linux/page-flags.h文件中，这部分也是在内核代码中用来判断页属性的关键部分。

## FLAGS定义

FLAGS是一个按照比特位来定义的属性集合，比如我们可以看一下定义从而大致了解一下这个FLAGS中都有哪些内容。

```
  enum pageflags {
  	PG_locked,		/* Page is locked. Don't touch. */
  	PG_referenced,
  	PG_uptodate,
  	PG_dirty,
  	PG_lru,
  	PG_active,
  	PG_workingset,
    ...
	__NR_PAGEFLAGS,
    ...
  }
```

整个FLAGS的长度为__NR_PAGEFLAGS，并且内核定义又了NR_PAGEFLAGS。这两者的值一致。

并且在page-flags-layout.h中，用下面的代码保证了这些神秘的东西加起来不会放不下。

```
#if ZONES_WIDTH + SECTIONS_WIDTH + NODES_WIDTH + KASAN_TAG_WIDTH + LAST_CPUPID_WIDTH \
	> BITS_PER_LONG - NR_PAGEFLAGS
#error "Not enough bits in page flags"
#endif
```

## FLAGS操作

有了定义，接下来就是如何操作。这里就是体现C语言博大精深的时刻了，其中对字段操作的宏定义总结如下：

    * PageXXX()
    * SetPageXXX()
    * ClearPageXXX()

这三个宏分别用来判断、设置、清除page->flags相应的属性。在内核代码中使用非常广泛，以后看到长成这样的，就可以到这个头文件中搜索。

# 定义FLAGS操作

虽然我们上面看到了，在代码中用的是SetPageXXX()/ClearPageXXX()来操作flags，但是因为这一套基本是同样的操作，所以代码中并没有给每个bit定义单独的Set/Clear。

而是通过两个宏来定义的(还有folio相关的操作，放在后面讲)

    * PAGEFLAG   原子操作
    * __PAGEFLAG 非原子操作

所以在page-flags.h文件中可以看到一串用PAGEFLAG()定义的内容。如

```c
    PAGEFLAG(Dirty, dirty, PF_HEAD)
    PAGEFLAG(LRU, lru, PF_HEAD)
    PAGEFLAG(Active, active, PF_HEAD)
```

而PAGEFLAG的定义是：

```
#define PAGEFLAG(uname, lname, policy)					\
	TESTPAGEFLAG(uname, lname, policy)				\
	SETPAGEFLAG(uname, lname, policy)				\
	CLEARPAGEFLAG(uname, lname, policy)
```

所以相当于是一套都定义好了，往下具体的实现是用test_bit/set_bit/clear_bit来做的。

另外值的注意的是，其中[TEST|SET|CLEAR]PAGEFLAG的定义都包含了对应FOLIO的定义。

## policy

在PAGEFLAG()定义中，有第三个参数policy。我们来看看它究竟是什么。

首先看看这个policy是怎么用的。

```
static __always_inline int Page##uname(const struct page *page)		\
{ return test_bit(PG_##lname, &policy(page, 0)->flags); }


static __always_inline void ClearPage##uname(struct page *page)		\
{ clear_bit(PG_##lname, &policy(page, 1)->flags); }
```

那这个policy究竟是什么呢？ 哦，是一大串宏定义:

```
#define PF_POISONED_CHECK(page) ({					\
		VM_BUG_ON_PGFLAGS(PagePoisoned(page), page);		\
		page; })
#define PF_ANY(page, enforce)	PF_POISONED_CHECK(page)
#define PF_HEAD(page, enforce)	PF_POISONED_CHECK(compound_head(page))
#define PF_NO_TAIL(page, enforce) ({					\
		VM_BUG_ON_PGFLAGS(enforce && PageTail(page), page);	\
		PF_POISONED_CHECK(compound_head(page)); })
#define PF_NO_COMPOUND(page, enforce) ({				\
		VM_BUG_ON_PGFLAGS(enforce && PageCompound(page), page);	\
		PF_POISONED_CHECK(page); })
#define PF_SECOND(page, enforce) ({					\
		VM_BUG_ON_PGFLAGS(!PageHead(page), page);		\
		PF_POISONED_CHECK(&page[1]); })
```

仔细看了一下，这个policy还有两个作用：

  * 检查传入的page是否符合要求
  * 将page转换为想要的对象

其中第二点在PF_HEAD/PF_NO_TAIL/PF_SECOND中会处理。这样最后检测/设置的flags就是对应head或者page[1]的flags，而不是传入的page的。

而作为检测手段，当第二个参数enforce为0的时候，某些check就不起作用了。而搜了一圈，只有TESTPAGEFLAG时，才会是0。

再详细解释一下其中几个定义：

  * PF_HEAD: 并没有检测，直接取了compound_head()返回，也不知道是何原因。不过使用它的几个flags也却是只能在head page。
  * PF_NO_TAIL: 和PF_HEAD一样，最后返回的也是head page。但区别在于PF_NO_TAIL会验证是否是head page。
  * PF_SECOND: 传入的page一定是head page，出来的是&page[1]。

PS： 最终这些PAGEFLAG，都要被替换成FOLIO_FLAG。

