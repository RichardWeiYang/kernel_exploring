建立反向映射的目的是为了能够查到某个页面当前在哪些进程中映射，相应的我们可以进行一些操作。比如clear, swap, migrate。

这个操作的框架基本是统一的，这里我们来学习一下相关的操作。

# rmap_walk[_locked]

使用反向映射的核心函数是

  - rmap_walk
  - rmap_walk_locked

通常他们在被调用的时候都会出现，作为持锁和不持锁的两个分支。 顺便我们再来看看哪些函数会调用这两个，从而了解一下内核中什么操作会需要用到反向映射。

  - page_idle_clear_pte_refs
  - folio_referenced
  - folio_mkclean
  - try_to_unmap
  - try_to_migrate
  - remove_migration_ptes

而上面两个函数展开，我们可以看到真正的核心其实是：

  - rmap_walk_anon()
  - rmap_walk_file()

也就是对匿名页和文件页的遍历，因为这两者的反向映射实现是不同的。但是，实际上我们展开后，看到最核心的逻辑又是一样的。

```
	[anon_]vma_interval_tree_foreach(vma, RB_ROOT,
			pgoff_start, pgoff_end) {
		unsigned long address = vma_address(vma, pgoff_start, nr_pages);

		VM_BUG_ON_VMA(address == -EFAULT, vma);
		cond_resched();

		if (rwc->invalid_vma && rwc->invalid_vma(vma, rwc->arg))
			continue;

		if (!rwc->rmap_one(folio, vma, address, rwc->arg))
			break
		if (rwc->done && rwc->done(folio))
			break
	}
```

根据给定的反向映射的RB_ROOT，遍历得到对应的vma。然后对应这个vma，分别执行

  - rwc->invalid_vma()
  - rwc->rmap_one()
  - rwc->done()

PS: rwc是struct rmap_walk_control，主要成员就是这三个回调函数。

事情真的就这么简单了吗？

## 如何定位反向映射

目前保存反向映射的数据结构是interval tree，所以在搜索interval tree的过程中，需要的参数是一个区域的范围。这就是我们看到了[pgoff_start, pgoff_end]。

```
	pgoff_start = folio_pgoff(folio) = folio->index; 
	pgoff_end = pgoff_start + folio_nr_pages(folio) - 1;
```

我们先来看一下index这个字段的含义：

```
 * @index: Offset within the file, in units of pages.  For anonymous memory,
 *    this is the index from the beginning of the mmap.
```

对文件页来讲，这个又和folio->mapping对上了。因为这个本来就是以文件中的offset来组织的。

对匿名页，这个值在__folio_set_anon()函数中设置。

```
	folio->index = linear_page_index(vma, address);
	             = vma->vm_pgoff + (address - vma->vm_start) >> PAGE_SHIFT;
```

其中，address是发生page fault时，这个页面对应的起始地址。简单来讲，这个index就是在进程地址空间的用户态地址。

## 用户态地址

只是在反向映射中找到vma还不够，接下来是要对vma中对应的地址空间做操作。在访问进程对应的页表前，我们先要确定的是我们究竟要访问的是进程中的哪个地址。这个就交给了vma_address()处理。

```
/**
 * vma_address - Find the virtual address a page range is mapped at
 * @vma: The vma which maps this object.
 * @pgoff: The page offset within its object.
 * @nr_pages: The number of pages to consider.
 *
 * If any page in this range is mapped by this VMA, return the first address
 * where any of these pages appear.  Otherwise, return -EFAULT.
 */
static inline unsigned long vma_address(const struct vm_area_struct *vma,
		pgoff_t pgoff, unsigned long nr_pages)
{
	unsigned long address;

	if (pgoff >= vma->vm_pgoff) {
		address = vma->vm_start +
			((pgoff - vma->vm_pgoff) << PAGE_SHIFT);
		/* Check for address beyond vma (or wrapped through 0?) */
		if (address < vma->vm_start || address >= vma->vm_end)
			address = -EFAULT;
	} else if (pgoff + nr_pages - 1 >= vma->vm_pgoff) {
		/* Test above avoids possibility of wrap to 0 on 32-bit */
		address = vma->vm_start;
	} else {
		address = -EFAULT;
	}
	return address;
}
```

其中第一个if判断中的内容是比较清除的，这个正好是匿名页设置index的逆操作。后面两个不太清楚，我偷懒了。。。

# page_vma_mapped_walk()

当我们通过反向映射找到对应的vma后，我们还需要在这个vma中对应的address地址中判断是否确实有我们要找的页面。（估计是因为有时候单个页面被unmap或者cow后，虽然还在反向映射里，但实际页面已经不是了。）page_vma_mapped_walk()函数就是用来做这个的。而且通常是在rwc->rmap_one()这个回调函数中使用。

在继续研究page_vma_mapped_walk()之前，我们先看看一个结构体struct page_vma_mapped_walk。（连名字都一样。。。）

```
#define DEFINE_FOLIO_VMA_WALK(name, _folio, _vma, _address, _flags)	\
	struct page_vma_mapped_walk name = {				\
		.pfn = folio_pfn(_folio),				\
		.nr_pages = folio_nr_pages(_folio),			\
		.pgoff = folio_pgoff(_folio),				\
		.vma = _vma,						\
		.address = _address,					\
		.flags = _flags,					\
	}
```

其实就只准备了一下后面处理的信息。按照我的理解，page_vma_mapped_walk()函数会在一次rmap_one()中调用多次，因为nr_pages可能不为1。这样要找到映射folio所有pte，就需要多走几步。

完整看下来，值得注意的是这里检测的是PTE对应的pfn是不是在[pfn, pfn + nr_pages)这个范围内。
