在page结构体中，针对页的使用情况有两个重要的值。(名字使用了老版本的命名。)

  * page.count
  * page.mapcount

在[Transparent huge page reference counting][1]文章中，有一段对这两个值含义的描述。

```
A reference count tracks the number of users an object (such as a page in memory)
has, allowing the kernel to determine when the object is free and can be deleted.
There are actually two types of reference counts for a normal page.

The first, stored in the _count field of struct page, is the total number of
references held to the page.
The second, kept in _mapcount, is the number of page table entries referring to this page.

A page-table mapping is a reference, so every such reference counted in _mapcount
is also tracked in _count; the latter should thus always be greater than or equal
to the former. Situations where _count can exceed _mapcount include pages mapped
for DMA and pages mapped into the kernel's address space with a function like
get_user_pages(). Locking a page into memory with mlock() will also increase _count.
The relative value of these two counters is important; if _count equals _mapcount,
the page can be reclaimed by locating and removing all of the page table entries.

But if _count is greater than _mapcount, the page is "pinned" and cannot be
reclaimed until the extra references are removed.
```

在简单的例子中，这两个值的使用和含义十分清楚。尤其是_mapcount这个值就是表示了当前的页被映射到了多少进程空间。

但是这一切的安宁被透明大页(THP)的到来给打破了。

比如说在当前内核中(v5.4)有这么一个函数：

```
int total_mapcount(struct page *page)
{
	int i, compound, ret;

	VM_BUG_ON_PAGE(PageTail(page), page);

	if (likely(!PageCompound(page)))
		return atomic_read(&page->_mapcount) + 1;

	compound = compound_mapcount(page);
	if (PageHuge(page))
		return compound;
	ret = compound;
	for (i = 0; i < HPAGE_PMD_NR; i++)
		ret += atomic_read(&page[i]._mapcount) + 1;
	/* File pages has compound_mapcount included in _mapcount */
	if (!PageAnon(page))
		return ret - compound * HPAGE_PMD_NR;
	if (PageDoubleMap(page))
		ret -= HPAGE_PMD_NR;
	return ret;
}
```

请问，你能告诉我这都算出来的是个什么吗？

当我第一次看到这个函数定义的时候，内心是崩溃的。这都是什么鬼，一会儿加还一会儿减，一会儿又直接返回。

所以要理解这些代码，一切还要从头来过。

# 页表项的几种可能以及遇到的问题

要理解上述代码，还要先从页表项的可能性说起。

现代计算机系统基本都包含了虚拟内存管理单元--页表。它负责完成的是从虚拟地址到物理地址之间的转换。总的来说，在页表中可以映射两种页面：

  * PTE page: 4k on x86
  * PMD page: 2M on x86

注：据说还能有PUD page，在这里为了简单先略过。

对于PTE page来说，一个4k的页面对应了页表中的一项。而对于PMD page来说，一个2M的页面对应了页表中的一项。

这一切看似没有什么区别，但是仔细往下看却发现了一些需要解决的问题。 原因在于在内核中，每一个4k的页面都对应着一个页结构体作为元数据描述符。

> 那么问题来了，当我想要查询某一个4k空间是否被映射到某个进程的时候，我是查看这个4k空间(pte leve)直接对应的页结构体，还是包含这个4k空间的2M页(pmd level)对应的页结构体？

这个问题在内核中用了一个比较直接的方法来解决 -- compound page。也就是在内核中提出了复合页这个概念，来告诉你要询问的个段空间是要当作pte来看待呢，还是当作pmd来看待。

如果是compound page，那么相关的信息会保存在这个复合页的第一个页(PageHead)结构体中。对compound page的描述请参考[页分配器][2]一章相应小节。

为此，内核中分别有

  * hugetlb, PMD level
  * anonymous page, pte level
  * file page, pte level

来满足这两种页表项的可能。

如果世界是如此美好，那么也就没有这篇文章了。因为除了刚才说的两种情况外，还有一种，或者更准确的说还有一些额外的变种情况会出现。

  * 同一块PMD区域，它既可以被当作PMD映射，也可以当作PTE映射

静态得看，一个页表项确实只能是PTE page或者PMD page。但是从内存本身的角度来看，同一块PMD level内存，它可能是被映射成PMD，也可能在另一个空间被部分映射成PTE。

在这种情况下，之前仅仅使用compound page就不够了。因为哪怕是compound页，其中的部分也可能被映射为pte。那此时该怎么办呢？

这个问题也就是内核中要实现透明大页(THP)所要解决的问题。而为了解决这个问题，内核社区付出了巨大的努力。

# 第一个版本 -- 不成功，便成仁

内核社区第一次进入主线的THP实现发生在2011年初。

正如[Transparent huge page reference counting][1]描述的：

```
In current kernel, a specific 4KB page can be treated as an individual
page, or it can be part of a huge page, but not both. If a huge page must
be split into individual pages, it is split completely for all users, the
compound page structure is torn down, and the huge page no longer exists.
```

这个版本的实现比较粗暴。对于一块PMD page，要么就是当作PMD映射，要么就是当作PTE映射，不能有同时存在的情况。所以在上一小节中遇到的问题**这个页究竟是PMD的一部分，还是一个单独的PTE**，在第一个版本中被消灭了。两者不可兼得

那么我们来看看在这个版本中具体发生了什么变化。

在这个版本中有两个映射关系相关的重要提交：

  * 918070634448 2011-01-13 thp: alter compound get_page/put_page
  * 71e3aac0724f 2011-01-13 thp: transparent hugepage core

我们一个个来看，首先是第一个提交thp: alter compound get_page/put_page中主要改变了_count的计算。

改变前：

```
static inline void get_page(struct page *page)
{
	page = compound_head(page);
	VM_BUG_ON(atomic_read(&page->_count) == 0);
	atomic_inc(&page->_count);
}
```

改变后：

```
static inline void get_page(struct page *page)
{
	/*
	 * Getting a normal page or the head of a compound page
	 * requires to already have an elevated page->_count. Only if
	 * we're getting a tail page, the elevated page->_count is
	 * required only in the head page, so for tail pages the
	 * bugcheck only verifies that the page->_count isn't
	 * negative.
	 */
	VM_BUG_ON(atomic_read(&page->_count) < !PageTail(page));
	atomic_inc(&page->_count);
	/*
	 * Getting a tail page will elevate both the head and tail
	 * page->_count(s).
	 */
	if (unlikely(PageTail(page))) {
		/*
		 * This is safe only because
		 * __split_huge_page_refcount can't run under
		 * get_page().
		 */
		VM_BUG_ON(atomic_read(&page->first_page->_count) <= 0);
		atomic_inc(&page->first_page->_count);
	}
}
```

这个改动看出，当我们需要获得一个PTE页，这个计数不仅会记录在本页结构体中，也会记录在compound页的第一个页结构体中。

接着我们看看第二个提交：thp: transparent hugepage core。

这个提交中的关键是函数__split_huge_page_refcount()。

```
__split_huge_page_refcount(), split the thp into normal pages
    atomic_sub(page_tail->_count, page->_count)
    atomic_add(page_mapcount(page) + 1, page_tail->_count)
    page_tail->_mapcount = page->_mapcount
    put_page(page_tail)
```

这个函数发生在将要PMD page退化成PTE page的时候。也就是要将之前当作PMD page映射的页，分解成一个个PTE page映射的页。

为了更好的理解上述这段代码，让我们来推演一下一个PMD page中间有一页被获得的情况下分解时的过程。（用两个页在做模拟，下同。）

```
a. new thp with __do_huge_pmd_anonymous_page()

page[0] +-----------------------+
        |_count                 |  = 1            page refcount set to 1
        |_mapcount              |  = -1 -> 0      from -1 to 0
page[1] +-----------------------+
        |_count                 |  = 0
        |_mapcount              |  = -1
        +-----------------------+

b. just get_page(page[1]) for example

page[0] +-----------------------+
        |_count                 |  = 1  -> 2      add 1 to count page[1]
        |_mapcount              |  = 0
page[1] +-----------------------+
        |_count                 |  = 0  -> 1      add 1 for itself
        |_mapcount              |  = -1
        +-----------------------+

c. __split_huge_page_refcount()

page[0] +-----------------------+
        |_count                 |  = 2  -> 1      dec 1 for page[1]._count
        |_mapcount              |  = 0
page[1] +-----------------------+
        |_count                 |  = 1  -> 2      one for mapping, one for get_page()
        |_mapcount              |  = -1 -> 0      set to page[0]._mapcount
        +-----------------------+

d. put_pge(page[1])

Since page is not a compound page anymore, just decrease _count is enough.

page[0] +-----------------------+
        |_count                 |  = 1
        |_mapcount              |  = 0
page[1] +-----------------------+
        |_count                 |  = 2  -> 1      dec 1
        |_mapcount              |  = 0
        +-----------------------+
```

在这个版本中， 因为不支持PMD page同时呈现两种映射状态，所以采用的方式在退化成pte映射时，将_count和_mapcount复制到pte page即可。所以页结构体中的元数据也几乎没有什么重复使用。而且有意思的是函数page_mapcount()没有调整，这说明在当时的代码中是不会直接去询问一个PMD中的PTE的映射数值的。

好了这个版本采用了十分粗暴的方式，不能说解决，而是杜绝了一种情况的发生。虽然有点粗暴，但是毕竟THP可以用了。

# 第二个版本 -- 波粒二象性

这么简单粗暴的方式，内核社区当然是想要改进的。这不，Kirill在2016年初提出了新的方案。

这个方案中关键的提交有四个：

    * ddc58f27f9ee 2016-01-15 mm: drop tail page refcounting
    * 53f9263baba6 2016-01-15 mm: rework mapcount accounting to enable 4k mapping of THPs
    * eef1b3ba053a 2016-01-15 thp: implement split_huge_pmd()
    * e9b61f19858a 2016-01-15 thp: reintroduce split_huge_page()

在[Transparent huge page reference counting][1]中提到：

```
The fundamental change in Kirill's patch set is to allow a huge page to be
split in one process's address space, while remaining a huge page in any
other address space where it is found.
```

从此PMD页和PTE页就再也不是相互不兼容的了。

接着我们来看看究竟是对mapcount做了什么改动。

第一个变化来自 ddc58f27f9ee 2016-01-15 mm: drop tail page refcounting。这个提交将原先get_page的改动又恢复了回来。为了减少篇幅，我们直接看改动后的最后结果。

```
static inline void get_page(struct page *page)
{
	page = compound_head(page);
	/*
	 * Getting a normal page or the head of a compound page
	 * requires to already have an elevated page->_count.
	 */
	VM_BUG_ON_PAGE(atomic_read(&page->_count) <= 0, page);
	atomic_inc(&page->_count);
}

static inline void init_page_count(struct page *page)
{
	atomic_set(&page->_count, 1);
}

static inline void put_page(struct page *page)
{
	page = compound_head(page);
	if (put_page_testzero(page))
		__put_page(page);
}
```

改动后compound page的计数都算在了PageHead页上。

当然这个提交中的内容远不止这些。这样简化之后，许多相关的地方都要调整。不过所有的调整都是因为我们不再需要因为PageTail而调整PageHead的计数了。

清理完后，Kirill就提出了新的mapcout规则 53f9263baba6 2016-01-15 mm: rework mapcount accounting to enable 4k mapping of THPs。

在这个提交中关键的是两点：

    * compound_mapcount的引入
    * PageDoubleMap的引入

然我们来看看[Transparent huge page reference counting][1]中是怎么对这两个新成员做介绍的：

compound_mapcount的含义

```
The idea is to store separately how many times the page was mapped as
whole -- compound_mapcount.

Any time we map/unmap whole compound page (THP or hugetlb) -- we
increment/decrement compound_mapcount.  When we map part of compound
page with PTE we operate on ->_mapcount of the subpage.
```

Kirill用了一个新成员来记录compound page的映射次数，这样就是释放出mapcount来记录PTE映射了。

PageDoubleMap的含义

```
PageDoubleMap indicates that the compound page is mapped with PTEs as well
as PMDs.

This is required for optimization of rmap operations for THP: we can
postpone per small page mapcount accounting (and its overhead from atomic
operations) until the first PMD split.

For the page PageDoubleMap means ->_mapcount in all sub-pages is offset up
by one. This reference will go away with last compound_mapcount.
```

这里标明了PageDoubleMap是对计数的一个优化。减少了对PTE映射计数的操作。具体来说就是如果没有这个优化，当映射一整个PMD时，需要增加compound_mapcount和每个PTE page的mapcount。引入了这个标示后，我们可以把这个操作延后到第一次拆分PMD的时候。

了解了概念后，我们来看看代码层面的变化。

首先我们来看看映射计数在计算上的变化：

```
Before:

static inline int page_mapcount(struct page *page)
{
	VM_BUG_ON_PAGE(PageSlab(page), page);
	return atomic_read(&page->_mapcount) + 1;
}

After:

static inline int page_mapcount(struct page *page)
{
	int ret;
	VM_BUG_ON_PAGE(PageSlab(page), page);

	ret = atomic_read(&page->_mapcount) + 1;
	if (PageCompound(page)) {
		page = compound_head(page);
		ret += atomic_read(compound_mapcount_ptr(page)) + 1;
		if (PageDoubleMap(page))
			ret--;
	}
	return ret;
}
```

这里可以看到，在引入了compound_mapcount后，一个4k页的映射计数分散在了两个地方：mapcount和compound_mapcount。当然还有一个讨厌的PageDoubleMap。

而这个PageDoubleMap的设置到了eef1b3ba053a 2016-01-15 thp: implement split_huge_pmd()才出现。其中核心的一段代码是：

```
/*
 * Set PG_double_map before dropping compound_mapcount to avoid
 * false-negative page_mapped().
 */
if (compound_mapcount(page) > 1 && !TestSetPageDoubleMap(page)) {
  for (i = 0; i < HPAGE_PMD_NR; i++)
    atomic_inc(&page[i]._mapcount);
}

if (atomic_add_negative(-1, compound_mapcount_ptr(page))) {
  /* Last compound_mapcount is gone. */
  __dec_zone_page_state(page, NR_ANON_TRANSPARENT_HUGEPAGES);
  if (TestClearPageDoubleMap(page)) {
    /* No need in mapcount reference anymore */
    for (i = 0; i < HPAGE_PMD_NR; i++)
      atomic_dec(&page[i]._mapcount);
  }
}
```

这里我要提出一个我的想法。按照我的理解，不一定要在设置PageDoubleMap的时候给每个sub page计数加一。感觉也可以在最后完全退化到PTE映射的时候再加一。只是这么改动在性能上好像也没有什么改变。

最后我们用一张图来看看Kirill改动后拆分一个PMD时这些数字的变化过程。这个例子演示的过程如下：

    * 一个进程映射了一段PMD页
    * 经过两次fork后一共有三个进程映射到同一个PMD
    * 之后有两个进程想要退化成PTE映射。

```
a. new thp with __do_huge_pmd_anonymous_page() and two process fork

First process

page[0] +-----------------------+
        |_count                 |  = 0   -> 1     compound page refcount set to 1
        |_mapcount              |  = -1
page[1] +-----------------------+
        |_count                 |  = 0
        |_mapcount              |  = -1
        |compound_mapcount      |  = -1  -> 0     THP PMD mapcount
        +-----------------------+

b. another process map the same THP PMD after fork

page[0] +-----------------------+
        |_count                 |  = 1   -> 2     compound page refcount inc 1
        |_mapcount              |  = -1
page[1] +-----------------------+
        |_count                 |  = 0
        |_mapcount              |  = -1
        |compound_mapcount      |  = 0   -> 1     THP PMD mapcount inc 1
        +-----------------------+

c. 3rd process map the same THP PMD after fork

page[0] +-----------------------+
        |_count                 |  = 2   -> 3     compound page refcount inc 1
        |_mapcount              |  = -1
page[1] +-----------------------+
        |_count                 |  = 0
        |_mapcount              |  = -1
        |compound_mapcount      |  = 1   -> 2     THP PMD mapcount inc 1
        +-----------------------+

d. one process want to split THP PMD

Add HPAGE_PMD_NR - 1 to page[0]._count, since we could put_page() on each PTE page.

page[0] +-----------------------+
        |_count                 |  = 3   -> 3 + HPAGE_PMD_NR - 1
        |_mapcount              |  = -1
page[1] +-----------------------+
        |_count                 |  = 0
        |_mapcount              |  = -1
        |compound_mapcount      |  = 2
        +-----------------------+

Increase PTE page _mapcount to indicate PTE page is mapped in process.

page[0] +-----------------------+
        |_count                 |  = 3 + HPAGE_PMD_NR - 1
        |_mapcount              |  = -1  -> 0     count for PTE map for this process
page[1] +-----------------------+
        |_count                 |  = 0
        |_mapcount              |  = -1  -> 0     count for PTE map for this process
        |compound_mapcount      |  = 2
        +-----------------------+

Since the first time to set PageDoubleMap, increase _mapcount.

page[0] +-----------------------+
        |_count                 |  = 3 + HPAGE_PMD_NR - 1
        |_mapcount              |  = 0   -> 1     count for DoubleMap
page[1] +-----------------------+
        |_count                 |  = 0
        |_mapcount              |  = 0   -> 1     count for DoubleMap
        |compound_mapcount      |  = 2
        +-----------------------+

Decrease compount_mapcount of the THP PMD, since current process will map PTE

page[0] +-----------------------+
        |_count                 |  = 3 + HPAGE_PMD_NR - 1
        |_mapcount              |  = 1
page[1] +-----------------------+
        |_count                 |  = 0
        |_mapcount              |  = 1
        |compound_mapcount      |  = 2   -> 1     THP PMD mapcount dec 1
        +-----------------------+

e. another process want to split THP PMD

Add HPAGE_PMD_NR - 1 to page[0]._count, since we could put_page() on each PTE page.

page[0] +-----------------------+
        |_count                 |  = 3 + HPAGE_PMD_NR - 1 -> 3 + 2 * (HPAGE_PMD_NR - 1)
        |_mapcount              |  = 1
page[1] +-----------------------+
        |_count                 |  = 0
        |_mapcount              |  = -1
        |compound_mapcount      |  = 1
        +-----------------------+

Increase PTE page _mapcount to indicate PTE page is mapped in process.

page[0] +-----------------------+
        |_count                 |  = 3 + 2 * (HPAGE_PMD_NR - 1)
        |_mapcount              |  = 1   -> 2     count for PTE map for this process
page[1] +-----------------------+
        |_count                 |  = 0
        |_mapcount              |  = 1   -> 2     count for PTE map for this process
        |compound_mapcount      |  = 1
        +-----------------------+

Since not the first time to set PageDoubleMap, _mapcount remains.

Decrease compount_mapcount of the THP PMD, since current process will map PTE

page[0] +-----------------------+
        |_count                 |  = 3 + 2 * (HPAGE_PMD_NR - 1)
        |_mapcount              |  = 2
page[1] +-----------------------+
        |_count                 |  = 0
        |_mapcount              |  = 2
        |compound_mapcount      |  = 1   -> 0     THP PMD mapcount dec 1
        +-----------------------+

f. the last process want to split THP PMD

page[0] +-----------------------+
        |_count                 |  = 3 + 3 * (HPAGE_PMD_NR - 1)
        |_mapcount              |  = 2   -> 3     count for PTE map for this process
page[1] +-----------------------+
        |_count                 |  = 0
        |_mapcount              |  = 2   -> 3     count for PTE map for this process
        |compound_mapcount      |  = 0   -> -1    no THP PMD mapping now
        +-----------------------+

Since not the last time to clear PageDoubleMap, _mapcount dec by 1.

page[0] +-----------------------+
        |_count                 |  = 3 + 3 * (HPAGE_PMD_NR - 1)
        |_mapcount              |  = 3   -> 2     now each PTE map in 3 process
page[1] +-----------------------+
        |_count                 |  = 0
        |_mapcount              |  = 3   -> 2     now each PTE map in 3 process
        |compound_mapcount      |  = -1
        +-----------------------+

```

记住此时只是拆分了PMD，也就是可以让一个PMD在不同进程中看到不同的样子。此时还是保持了整个PMD页是一个compound page。

真正对整个PMD页的拆分在e9b61f19858a 2016-01-15 thp: reintroduce split_huge_page()引入。那我们就来看看真正拆分大页时映射计数的变化。

接着上面例子中的状态：

```
page[0] +-----------------------+
        |_count                 |  = 3 * HPAGE_PMD_NR
        |_mapcount              |  = 2
page[1] +-----------------------+
        |_count                 |  = 0
        |_mapcount              |  = 2
        |compound_mapcount      |  = -1
        +-----------------------+

Add PTE's mapcount + 1 to PTE's count. Something like offload the statistic.

page[0] +-----------------------+
        |_count                 |  = 3 * HPAGE_PMD_NR -> 3 * HPAGE_PMD_NR + 3
        |_mapcount              |  = 2
page[1] +-----------------------+
        |_count                 |  = 0   -> 3
        |_mapcount              |  = 2
        |compound_mapcount      |  = -1
        +-----------------------+

Release the mapcount from each subpage from head->_count.

page[0] +-----------------------+
        |_count                 |  = 3 * HPAGE_PMD_NR + 3 -> 3
        |_mapcount              |  = 2
page[1] +-----------------------+
        |_count                 |  = 3
        |_mapcount              |  = 2
        |compound_mapcount      |  = -1
        +-----------------------+
```

# 第三个版本 -- 文件透明大页

到上面的状态THP基本已经实现完了，除了文件透明大页。这部分的功能也是Kirill在2016年夏天实现的。

```
dd78fedde4b9 2016-07-26 rmap: support file thp
```

其实在我看来没有看出为什么文件透明大页和匿名透明大页之间有什么实现上的不同。但是在提交记录中有这么一句话：

```
 PG_double_map optimization doesn't work for file pages since lifecycle
 of file pages is different comparing to anon pages: file page can be
 mapped again at any time.
```

看来和lru还有关系，这点我还得继续好好研究了。好了，那我们来看看这个版本中带来的变化。

在这个版本，也就是上述提交中，重要变化的是这三个函数

```
page_add_file_rmap
total_mapcount
__page_mapcount
```

变化前：

```
void page_add_file_rmap(struct page *page)
{
	lock_page_memcg(page);
	if (atomic_inc_and_test(&page->_mapcount)) {
		__inc_zone_page_state(page, NR_FILE_MAPPED);
		mem_cgroup_inc_page_stat(page, MEM_CGROUP_STAT_FILE_MAPPED);
	}
	unlock_page_memcg(page);
}

int total_mapcount(struct page *page)
{
	int i, ret;

	VM_BUG_ON_PAGE(PageTail(page), page);

	if (likely(!PageCompound(page)))
		return atomic_read(&page->_mapcount) + 1;

	ret = compound_mapcount(page);
	if (PageHuge(page))
		return ret;
	for (i = 0; i < HPAGE_PMD_NR; i++)
		ret += atomic_read(&page[i]._mapcount) + 1;
	if (PageDoubleMap(page))
		ret -= HPAGE_PMD_NR;
	return ret;
}


int __page_mapcount(struct page *page)
{
	int ret;

	ret = atomic_read(&page->_mapcount) + 1;
	page = compound_head(page);
	ret += atomic_read(compound_mapcount_ptr(page)) + 1;
	if (PageDoubleMap(page))
		ret--;
	return ret;
}
```

变化后：

```
void page_add_file_rmap(struct page *page, bool compound)
{
	int i, nr = 1;

	VM_BUG_ON_PAGE(compound && !PageTransHuge(page), page);
	lock_page_memcg(page);
	if (compound && PageTransHuge(page)) {
		for (i = 0, nr = 0; i < HPAGE_PMD_NR; i++) {
			if (atomic_inc_and_test(&page[i]._mapcount))
				nr++;
		}
		if (!atomic_inc_and_test(compound_mapcount_ptr(page)))
			goto out;
	} else {
		if (!atomic_inc_and_test(&page->_mapcount))
			goto out;
	}
	__mod_zone_page_state(page_zone(page), NR_FILE_MAPPED, nr);
	mem_cgroup_inc_page_stat(page, MEM_CGROUP_STAT_FILE_MAPPED);
out:
	unlock_page_memcg(page);
}

int total_mapcount(struct page *page)
{
	int i, compound, ret;

	VM_BUG_ON_PAGE(PageTail(page), page);

	if (likely(!PageCompound(page)))
		return atomic_read(&page->_mapcount) + 1;

	compound = compound_mapcount(page);
	if (PageHuge(page))
		return compound;
	ret = compound;
	for (i = 0; i < HPAGE_PMD_NR; i++)
		ret += atomic_read(&page[i]._mapcount) + 1;
	/* File pages has compound_mapcount included in _mapcount */
	if (!PageAnon(page))
		return ret - compound * HPAGE_PMD_NR;
	if (PageDoubleMap(page))
		ret -= HPAGE_PMD_NR;
	return ret;
}

int __page_mapcount(struct page *page)
{
	int ret;

	ret = atomic_read(&page->_mapcount) + 1;
	/*
	 * For file THP page->_mapcount contains total number of mapping
	 * of the page: no need to look into compound_mapcount.
	 */
	if (!PageAnon(page) && !PageHuge(page))
		return ret;
	page = compound_head(page);
	ret += atomic_read(compound_mapcount_ptr(page)) + 1;
	if (PageDoubleMap(page))
		ret--;
	return ret;
}
```

这些变化的原因就是提交记录中描述的，PG_double_map没法作为映射计数的优化了。必须在每次映射时(page_add_file_rmap)增加每个PTE映射值。

值得一提的是total_mapcount和__page_mapcount的作用范围。

之前看代码的时候一值困惑这两个函数的区别，想着既然有了前者为什么还要有后者，后来才明白了。
  * total_mapcount是要将一个页面当作大页来看待的
  * 而__page_mapcount则是作为PTE映射来看的。

所以两者的实现会略有不同。

[1]: https://lwn.net/Articles/619738/
[2]: /mm/06-page_alloc.md
