# linear_page_index()

```
static inline pgoff_t linear_page_index(const struct vm_area_struct *vma,
					const unsigned long address)
{
	pgoff_t pgoff;
	pgoff = (address - vma->vm_start) >> PAGE_SHIFT;
	pgoff += vma->vm_pgoff;
	return pgoff;
}
```
其中vm_start/vm_pgoff的含义可以参考[vma单个vma的内容][1]中的解释。

对于文件映射，得到的就是对应地址在文件中的实际偏移。
对于匿名映射，得到的正常来说就还是address。

# 设置

```
    __handle_mm_fault
        vmf.pgoff = linear_page_index(vma, address)                   // 匿名：虚拟地址；文件：文件内偏移
        ...
        do_anonymous_page
            folio_add_new_anon_rmap
                __folio_set_anon
                    folio->index = linear_page_index(vma, address)    // 也就是对应的进程虚拟地址
        do_fault
	...
            filemap_fault
                index = vmf->pgoff
                __filemap_get_folio(, index, ) -> __filemap_get_folio_mpol(, index, )
                    filemap_add_folio(folio, index, )
                        __filemap_add_folio(folio, index, )
                            XA_STATE_ORDER(xas, &mapping->i_pages, index, folio_order(folio))
                            folio->index = xas.xa_index;              // 实际上就是文件内偏移
                                                                         不过会folio_order()对齐
```

[1]: /virtual_mm/05-vma.md
