在[使用][1]部分我们看到了hugetlb的两种使用方式：

  * 匿名映射
  * 文件映射

但是归根结底都是通过mmap的。

# 映射

我们从mmap的系统调用来看一下映射时发生了什么。

```
ksys_mmap_pgoff(addr, len, prot, flags, fd, pgoff)
    if (!(flags & MAP_ANONYMOUS))
        file = fget(fd)
        if (is_file_hugepages(file))                                 // hugetlb文件映射
            len = ALIGN(len, huge_page_size(hstate_file(file)))
    else if (flags & MAP_HUGETLB)                                    // 匿名hugetlb映射
        file = hugetlb_file_setup()

    vm_mmap_pgoff(file, addr, len, prot, flags, pgoff)
               -> do_mmap() -> mmap_region() -> __mmap_region()
        __mmap_new_vma()
            vma = vm_area_alloc(map->mm)
            __mmap_new_file(map, vma) -> mmap_file(file, vma)
                vfs_mmap(file, vma)
                    file->f_op->mmap(file, vma)
```

从这个流程中看到，不论是匿名还是文件映射，在内核中都认为是一个文件映射。（这点和共享文件有点像）

而文件映射会调用到文件自身的mmap函数做初始化。这个函数对于hugetlb来说就是 hugetlbfs_file_mmap().

```
hugetlbfs_file_mmap()
    vma_set_flags(vma, VMA_HUGETLB_BIT, VMA_DONTEXPAND_BIT)
    vma->vm_ops = &hugetlb_vm_ops
```

上面这个VMA_HUGETLB_BIT设置我找了半天，因为在下面缺页中断中判断是不是hugetlbfs的函数要用到。

```
is_vm_hugetlb_page(vma)
    is_vma_hugetlb_flags(&vma->flags)
        vma_flags_test(flags, VMA_HUGETLB_BIT)
```

# 缺页


[1]: /hugetlb/01-usage.md
