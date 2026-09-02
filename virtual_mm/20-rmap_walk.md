建立反向映射的目的是为了能够查到某个页面当前在哪些进程中映射，相应的我们可以进行一些操作。比如clear, swap, migrate。

这个操作的框架基本是统一的，这里我们来学习一下相关的操作, 也正好把相关知识点串起来。

# 反向映射的核心

说实话，我看了那么久代码，现在才理解过来，反向映射到底描述的是谁和谁之间的关系。

  **反向映射，按照地址空间绝对偏移排序了vma，然后通过地址空间绝对偏移找到对应vma并将其转换到对应进程的虚拟地址，并最后通过页表遍历找到映射了对应foloi的页表项**

所以，这个过程拆分成三个步骤：

  * anon_rmap_tree_insert(): 将vma按照地址空间绝对偏移插入到interval tree
  * rmap_walk() : 通过地址空间中的绝对地址找到映射了该区域的vma
  * page_vma_mapped_walk(): 找到对应的vma后，算出对应进程的虚拟地址，最后通过虚拟地址找到映射了该folio的页表项

至此，我们就来看看细节吧。

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
	pgoff_start = folio_pgoff(folio) = folio->index; 
	pgoff_end = pgoff_start + folio_nr_pages(folio) - 1;
	[anon_]vma_interval_tree_foreach(vma, RB_ROOT,
					 pgoff_start, pgoff_end) {
		unsigned long address = vma_anon_address(vma, pgoff_start, nr_pages);
		...or..
		unsigned long address = vma_filebacked_address(vma, pgoff_start, nr_pages);

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

关于folio->index的设置和含义，在[folio->index][1]中有详细解释。这里要强调的一点是，这个index保存的是所在地址空间的绝对偏移（包括文件映射和匿名映射）。

到这里本来应该就结束了，但是我发现我少了一部分的背景，少了一部分应该在[anon rmap][2]中学习的知识。

  > 我们在反向映射的interval tree中插入的元素是按照什么下标来插入的

这个问题得回到，以匿名反向映射为例，匿名反向映射过程中插入一个元素的过程。

```
static pgoff_t avc_start_pgoff(struct anon_vma_chain *avc)
{
	return vma_start_anon_pgoff(avc->vma);
}

static pgoff_t avc_last_pgoff(struct anon_vma_chain *avc)
{
	return vma_last_anon_pgoff(avc->vma);
}

INTERVAL_TREE_DEFINE(struct anon_vma_chain, rb, pgoff_t, rb_subtree_last,
		     avc_start_pgoff, avc_last_pgoff,
		     static, __anon_rmap_tree)

void anon_rmap_tree_insert(struct anon_vma_chain *avc,
			   struct anon_vma *anon_vma)
{
#ifdef CONFIG_DEBUG_VM_RB
	avc->cached_vma_start = avc_start_pgoff(avc);
	avc->cached_vma_last = avc_last_pgoff(avc);
#endif
	__anon_rmap_tree_insert(avc, &anon_vma->rb_root);
}
```

从中可以看出，插入到反向映射interval tree的下标是**vma位于地址空间的绝对偏移**.

所以在查找的时候，同样使用是绝对偏移的folio->index就呼应了。

# page_vma_mapped_walk()

当我们通过反向映射找到对应的vma后，我们还需要在这个vma中对应的address地址中判断是否确实有我们要找的页面。（估计是因为有时候单个页面被unmap或者cow后，虽然还在反向映射里，但实际页面已经不是了。）page_vma_mapped_walk()函数就是用来做这个的。而且通常是在rwc->rmap_one()这个回调函数中使用。

但是在查看页表之前，有一件事需要先处理： 

## 计算进程中的虚拟地址

进程中虚拟地址的计算由函数vma_anon_address(vma, pgoff, nr_pages) / vma_filebacked_address()计算。

这部分解释在[vma][3]和[folio->index][1]中分别做了解释。

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

[1]: /virtual_mm/21-folio_index.md
[2]: /virtual_mm/06-anon_rmap_usage.md
[3]: /virtual_mm/05-vma.md
