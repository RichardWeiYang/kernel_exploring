内存分类的方法有很多，其中有一种是怎么也绕不过去的分类方法：

  * 匿名页
  * 文件页

从前一直回避文件页的缺页处理，这次要好好梳理一下这两种缺页处理的区别。

# mmap的预备知识

不论是匿名页还是文件页，都是通过mmap映射到进程空间，然后在访问时触发缺页中断。所以缺页中断中采用和中处理方式，在mmap时就已经定好了。

这部分内容在[私有和共享映射][1]有较为详细的描述。

# 缺页中断的区分

我们来看缺页中断中，是如何将文件页和匿名页区分处理的。

大页处理

```
static inline vm_fault_t create_huge_pmd(struct vm_fault *vmf)
{
	struct vm_area_struct *vma = vmf->vma;
	if (vma_is_anonymous(vma))
		return do_huge_pmd_anonymous_page(vmf);
	if (vma->vm_ops->huge_fault)
		return vma->vm_ops->huge_fault(vmf, PMD_ORDER);
	return VM_FAULT_FALLBACK;
}
```

PTE 处理

```
static vm_fault_t do_pte_missing(struct vm_fault *vmf)
{
	if (vma_is_anonymous(vmf->vma))
		return do_anonymous_page(vmf);
	else
		return do_fault(vmf);
}
```

从这两者我们可以看到，区分的重要判断是vma_is_anonymous()。

而从[私有和共享映射][1]中的分析，我们看到vma_is_anonymous()判断的是vma->vm_ops是否为空。而在四种组合中，只有私有匿名映射，vm_ops才为空。

# 文件页缺页中断

匿名页缺页中断比较熟悉了，这里我们重点看看文件页缺页中断。

```
do_fault
    if (!(vmf->flags & FAULT_FLAG_WRITE))
        do_read_fault()
    if (!(vma->vm_flags & VM_SHARED))
        do_cow_fault()
    else
        do_shared_fault()
```

到这里，我们终于能够理解四种映射情况对应的缺页中断处理了。

  * 私有匿名映射：do_anonymous_page()
  * 共享匿名映射：do_fault()
  * 私有文件映射：do_fault()
  * 共享文件映射：do_fault()

因为其余三者都有vm_ops，所以走的路径都是一样的。但是对操作类型：读/写，以及映射类型：共享/私有又做了区分。下面我们来仔细分析一下。

## 读

我们先来看看读。

```
do_read_fault()
    __do_fault()
        vma->vm_ops->fault() -> filemap_fault()                    // refcount = nr_pages + 1
            folio = filemap_get_folio()
            vmf->page = folio_file_page(folio, index)
    finish_fault()
        folio_ref_add(folio, nr_pages - 1)                         // refcount = nr_pages * 2
        set_pte_range()
            folio_add_file_rmap_ptes(folio, nr_pages)              // mapcount = nr_pages
```

读很正常，这三种情况都会从文件中读取。

## 私有写

但是写就分为了私有写和共享写。当然私有写，只可能发生在私有文件映射。

有几种情况会走到私有写：

  * 原先这个页面从来没有读过，直接写了
  * 原先这个页面读取了，第一次写
  * 原先这个页面写过了，再次写

这几种情况会有区别吗？我们带着这些疑问展开代码。

```
do_cow_fault
    vmf_anon_prepare(vmf)
    folio = folio_prealloc()                                       // refcount = 1
    vmf->cow_page = &folio->page
    __do_fault()
        vma->vm_ops->fault() -> filemap_fault()
            folio = filemap_get_folio()
            vmf->page = folio_file_page(folio, index)
    copy_mc_user_highpage(vmf->cow_page, vmf->page...)
    finish_fault()
        folio_ref_add(folio, nr_pages - 1)                         // refcount = nr_pages
        set_pte_range()
            folio_add_new_anon_rmap(folio, )                       // mapcount = nr_pages
```

首先看到的是，执行了vmf_anon_prepare()也就是准备了匿名反向映射的数据结构。但是这也满神奇的。因为一个filemap的vma，竟然还有anon_vma。

然后在finish_fault()->set_pte_range()中将folio->mapping设置为匿名映射。这样folio_test_anon()才会返回是匿名页。

中间通过filemap_fault()把原有的文件内容读取出来，并复制到了新分配的页面。这样就表示，私有写是在原有文件内容基础上的修改。

这样就回答了上面三个问题中的两个：从来没有度过或者是哪怕之前读过了，写的时候都是在原有文件内容基础上。

但是如果在同一页面的不同位置又写了一次呢？关键知识点：此时不会再进入缺页中断，因为第一次写时，页面被标记为可写。

至此，我们回答完了自己提出的三个问题。

## 共享写

现在就剩下一个共享写的缺页情况了。

```
do_shared_fault
    __do_fault()
    vma->vm_ops->page_mkwrite()
    finish_fault()
```

这么看好像挺简单的，不过我们目前为止一直没有打开背后真正的大boss -- filemap_fault()。

# filemap_fault

在文件映射的缺页中断中，我们看到的是__do_fault()函数被调用。但是展开看实际调用的是vma->vm_ops->fault()。

大致扫了一眼，这个指针一般指向的是filemap_fault。就算不直接是filemap_fault，也会间接调用filemap_fault。所以文件缺页的关键处理绕不开这个函数。

下面只是一个粗略的大致流程。

```
filemap_fault
    file = vmf->vma->vm_file
    mapping = file->f_mapping
    index = vmf->pgoff
    folio = filemap_get_folio(mapping, index)                  // in pagecache?

    // if in pagecache
    do_async_mmap_readahead()
    // if not in pagecache
    fpin = do_sync_mmap_readahead()                            // sync readahead
        fpin = maybe_unlock_mmap_for_io(vmf, )
        page_cache_sync_ra(&ractl, ra->ra_pages)


    folio = __filemap_get_folio(mapping, index, FGP_CREAT|FGP_FOR_MMAP, vmf->gfp_mask)
        folio = filemap_get_entry(mapping, index)              // search in mapping

        // if not in pagecache, allocate one
        folio = filemap_alloc_folio(alloc_gfp, order)          // alloc and set refcount to 1
        filemap_add_folio(mapping, folio, index, gfp)          // add into pagecache
            __folio_set_locked(folio)
            __filemap_add_folio(mapping, folio, index, ...)
                folio_ref_add(folio, nr_pages)                 // refcount = 1 + nr_pages
                folio->mapping = mapping;
                folio->index = xas.xa_index;
            folio_add_lru(folio)

    vmf->page = folio_file_page(folio, index)

finish_fault
    folio_ref_add(folio, nr_pages - 1)                         // refcount = 2 * nr_pages
    set_pte_range()
        folio_add_file_rmap_ptes(folio, nr_pages)              // mapcount = nr_pages
```

## refcount和mapcount变化

先来看看，如果是一个新的页面添加到pagecache时计数的变化：

  * filemap_alloc_folio:   1
  * __filemap_add_folio:  +nr_pages
  * finish_fault:         +nr_pages-1

所以最后计数值是2*nr_pages，而在set_pte_range()中增加的mapcount是nr_pages。这样正好符合folio_expected_ref_count()的预期。

## do_sync_mmap_readahead()

同步预读取也会分配folio，并添加到pagecache（同样是通过filemap_add_folio）。

所以__filemap_get_folio()拿不到内存，感觉是小概率事件。

[1]: /virtual_mm/17-map_private_shared.md
