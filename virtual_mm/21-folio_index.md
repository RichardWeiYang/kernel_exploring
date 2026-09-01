Folio结构中有一个index字段，这个东西貌似定义了这个folio被映射到地址空间的一个位值。但是我有点不确定具体含义，来研究一下。

# folio->index设置

## 文件映射

```
    __handle_mm_fault
        vmf.pgoff = linear_page_index(vma, address)                        // 匿名：虚拟地址；文件：文件内偏移
        ...
        do_fault
	...
            filemap_fault
                index = vmf->pgoff
                __filemap_get_folio(, index, ) -> __filemap_get_folio_mpol(, index, )

                    index = mapping_align_index(mapping, index);           // index对齐文件系统最小order
                    order = __ffs(index);                                  // order是从index对齐地址i中获取的
                    folio = filemap_alloc_folio(, order, );                // 再用order去分配folio,所以隐含了index和order对齐的信息

                    filemap_add_folio(folio, index, )
                        __filemap_add_folio(folio, index, )
                            XA_STATE_ORDER(xas, &mapping->i_pages, index, folio_order(folio))
                            folio->index = xas.xa_index;                   // 实际上就是文件内偏移
```

从上面的逻辑看出，folio->index就是这个folio在文件中映射的地址偏移。也就是通过folio->index是可以从filemap中查到这个folio的。

## 匿名映射

```
    __handle_mm_fault
        vmf.pgoff = linear_page_index(vma, address)                        // 匿名：虚拟地址；文件：文件内偏移
        ...
        do_anonymous_page
            folio_add_new_anon_rmap
                __folio_set_anon(folio, vma, address, exclusive)
                    folio->index = linear_anon_page_index(vma, address)    // 也就是对应的进程虚拟地址
```

这里很有意思，虽然缺页中断最开始计算了linear_page_index()，但最后匿名页使用的是linear_anon_page_index()。

我们从下面的解释中可以看到，这时index实际上就是这个匿名页在进程中虚拟地址的偏移。

## linear_page_index(vma, addr) / linear_anon_page_index(vma, addr)

```
static inline pgoff_t linear_page_delta(const struct vm_area_struct *vma,
					const unsigned long address)
{
	return (address - vma->vm_start) >> PAGE_SHIFT;
}

static inline pgoff_t linear_page_index(const struct vm_area_struct *vma,
					const unsigned long address)
{
	return linear_page_delta(vma, address) + vma_start_pgoff(vma);
}

static inline pgoff_t __linear_anon_page_index(const struct vm_area_struct *vma,
		const unsigned long address)
{
	return linear_page_delta(vma, address) + vma_start_anon_pgoff(vma);
}
```

其中vm_start/vma_start_pgoff/vma_start_anon_pgoff的含义可以参考[vma单个vma的内容][1]中的解释。

对于文件映射，得到的就是对应地址在文件中的实际偏移。
对于匿名映射，得到的是在进程虚拟地址空间中的偏移。

**所以folio->index中存放的是在某个地址空间的偏移量**

但是我仔细一下，与其说是偏移量，实际上就是一个绝对量。

  * 对于文件映射，folio->index是可以直接从xarray中找到内存的下标
  * 对于匿名映射，folio->index是进程空间的虚拟地址（remap前）

# folio->index的使用

设置了index，最终是为了使用。这里我们终于可以解答埋在[vma][1]中关于vma_anon_address/vma_filebacked_address的疑问了。

这两个函数的第二个参数，通常是通过 page_pgoff(folio, page) / folio_pgoff(folio) 获得的。

```
static inline pgoff_t page_pgoff(const struct folio *folio,
		const struct page *page)
{
	return folio->index + folio_page_idx(folio, page);
}

static inline pgoff_t folio_pgoff(const struct folio *folio)
{
	return folio->index;
}
```

那我们以文件映射为例，展开看一下这个流程：

为了简化，我们就看folio起始页的地址计算。

先看folio_pgoff(folio)的值：

```
  folio_pgoff(folio) -> folio->index
  -> linear_page_delta(vma, address) + vma_start_pgoff(vma)
  -> (address - vma->vm_start) >> PAGE_SHIFT + vma_start_pgoff(vma)
```

然后再代入vma_filebacked_address()

```
  vma_filebacked_address(vma, folio_pgoff(folio), nr_pages)
  -> __vma_address(vma, folio_pgoff(folio), vma_start_pgoff(vma), nr_pages)
  -> vma->vm_start + (folio_pgoff(folio) - vma_start_pgoff(vma)) << PAGE_SHIFT
  -> vma->vm_start + (address - vma->vm_start)
  -> address
```

这么看，算出来就正好是缺页中断时，缺页的那个虚拟地址了。

## 文件映射remap后

我们假设，发生缺页中断后我们移动了vma，那这个时候再计算vma_filebacked_address()会是什么结果？

首先我们确认的是对应的folio->index是不会改变的，因为这时有可能有多个进程共享映射了这个页面，而且这个folio->index是在文件中所在位值的绝对量。

我们假设vma往后搬移了X，此时vma->vm_start_new = vma->vm_start + X。代入到上面的流程:

```
  vma_filebacked_address(vma, folio_pgoff(folio), nr_pages)
  -> __vma_address(vma, folio_pgoff(folio), vma_start_pgoff(vma), nr_pages)
  -> vma->vm_start_new + (folio_pgoff(folio) - vma_start_pgoff(vma)) << PAGE_SHIFT
  -> vma->vm_start_new + (address - vma->vm_start)
  -> address + (vma->vm_start_new - vma->vm_start)
  -> address + X
```

所以算出来的值正好是右移后的新的虚拟地址。

## 匿名映射remap后

前面我们看了文件映射，现在来看看匿名映射。

先看folio_pgoff(folio)的值：

```
  folio_pgoff(folio) -> folio->index
  -> linear_page_delta(vma, address) + vma_start_anon_pgoff(vma)
  -> (address - vma->vm_start) >> PAGE_SHIFT + vma_start_anon_pgoff(vma)
```

然后再代入vma_anon_address()

```
  vma_anon_address(vma, folio_pgoff(folio), nr_pages)
  -> __vma_address(vma, folio_pgoff(folio), vma_start_anon_pgoff(vma), nr_pages)
  -> vma->vm_start + (folio_pgoff(folio) - vma_start_anon_pgoff(vma)) << PAGE_SHIFT
  -> vma->vm_start + (address - vma->vm_start)
  -> address
```

所以和文件映射一样，这里计算出的是folio映射到进程虚拟地址空间的虚拟地址。

而发生remap后，和文件映射一样，只有vm_start变成了新值，其他的值都没有变。此时也会根据移动的差值得到新的虚拟地址。

[1]: /virtual_mm/05-vma.md
