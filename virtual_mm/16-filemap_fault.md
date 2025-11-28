内存分类的方法有很多，其中有一种是怎么也绕不过去的分类方法：

  * 匿名页
  * 文件页

这里我们就来看看文件页是怎么load到内存的。

# mmap的预备知识

不论是匿名页还是文件页，都是通过mmap映射到进程空间，然后在访问时触发缺页中断。所以缺页中断的行为，在mmap时就已经定好了。

下面列出关心的内容：

```
__mmap_region(file, )
    call_mmap_prepare()
    __mmap_new_vma()
        __mmap_new_file_vma()
            vma->vm_file = get_file(file)
        vma_link_file()                           // 把vma加到file->f_mapping
    set_vma_user_defined_fields()
        vma->vm_ops = map->vm_ops;
```

这里主要设置了vma的两个成员：

  * vm_file
  * vm_ops

对应到文件系统上，我们举个例子：

  * vm_file->f_op:  就是ext4_file_operations
  * vm_ops:         就是ext4_file_vm_ops

# 缺页中断

这里我们只看对文件页PTE层的中断处理do_fault，并且以其中只读缺页为例。

```
do_read_fault()
    __do_fault()
        vma->vm_ops->fault() -> filemap_fault()
    finish_fault()
        folio_ref_add(folio, nr_pages - 1)
        set_pte_range()
```

## filemap_fault
