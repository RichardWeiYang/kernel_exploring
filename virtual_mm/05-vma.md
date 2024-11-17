既然每个进程有自己的虚拟地址空间，那么就需要有东西把它管理起来。在内核中，这个数据结构就是vm_area_struct，简称vma。

# 单个vma的内容

虽然一叶障目是不太好的，但是为了了解整个vma树，我们还是得从vma结构体这片叶子说起。

下面是我简化后的一个vma结构体的重要成员。

```
    vm_area_struct
    +--------------------------------+
    |vm_start, vm_end                |    the range we cover
    |   (unsigned long)              |
    |                                |
    |vm_file                         |
    |   (struct file*)               |
    |vm_pgoff                        |    offset in PAGE_SIZE
    |   (unsigned long)              |
    +--------------------------------+
    |vm_flags                        |    VM_READ/WRITE/EXEC
    |   (unsigned long)              |
    |vm_page_prot                    |    access PTE permission of this VMA calculated from vm_flags
    |   (pgprot_t)                   |    _PAGE_PRESENT/RW/ACCESSED/DIRTY
    |                                |
    +--------------------------------+
    |vm_ops                          |
    |  (struct vm_operations_struct*)|
    +--------------------------------+
```

让我来一一解释一下：

  * 接下来是当前vma在虚拟空间中所占的位置，以及对应的文件及在文件中的位置
  * 最后几个规定了vma的属性，读/写/执行

这么一看好像也挺简单的了。

# 常用的API

学习一个数据结构，我们就要学习一些对这个数据结构基本的操作方式。我大致将这些操作分成几个类别：

  * 分配/释放
  * 查找
  * 插入/删除
  * 拆分/整合

## 分配/释放

分配和释放的函数分别是

  * vm_area_alloc
  * vm_area_free

实际是调用的kmem_cache的kernel接口来实现的。

除了这两个，还有一个函数vm_area_dup，用来复制一个vma出来。这个在进程fork的时候发挥作用。

除此之外，还有一个点需要注意的是，新分配出来的vma结构体需要通过vma_init()函数来初始化。想要了解vma诞生时候的样子，可以看一眼这个函数。

分配和释放其实比较简单。学习么，我们就从简单的开始，让自己渐入佳境。

## 查找

查找的函数也有两个

  * find_vma(mm, addr)

但是用途上略有差别，前者是只为了查找来的，而后者是为了插入做准备的。

而且find_vma函数和我们普通的查找还不一样，我们一般想象的是这个函数会返回一个vma，这个vma是覆盖地址addr的。但是你仔细看这个函数，其实返回的vma表示的是第一个满足addr < vm_end条件的。也就是这个addr可能并不在任何vma空间内。


## 插入/删除



## 拆分/整合

拆分和整合在内核中主要有两个大的函数来处理：

  * split_vma

在这两个暴露在外的函数后面，有一个非常重要的公共函数__vma_adjust。

### vma_adjust

vma_adjust和其变体__vma_adjust将被众多函数调用，在不同的情况下又展现出不同的逻辑。该函数有多个参数，而区别这个函数行为的主要是最后两个函数insert和expand。

既然有两个纬度，那么就有四种组合。而根据这几种组合我们来看看究竟对应的是哪种情况：

  insert/expand:           caller

  non-NULL/NULL            split_vma
  NULL/non-NULL            vma_merge
  NULL/NULL                mremap/shift_arg_pages

可以看出，当前只有三种组合会出现，而insert/expand同时非空的情况是没有的。对于第三种全是空的情况我们暂且不提，主要来看看刚才我们提到的第一二种情况 split_vma 和 vma_merge。

### split_vma

split_vma相对而言是比较简单的一个函数，其目的就是将原有的一个vma拆分成两个。这样做是为了改变其中一个vma的某些属性，比如在函数madvise_behavior()中需要做的。

split_vma在调用vma_adjust前会先准备好一个新的vma结构体，并且处理好它的vm_start/vm_end/vm_pgoff。然后将它作为参数传给vma_adjust，对应的情况是 insert = non-NULL, expand = NULL。在这种情况下__vma_adjust()就变得简单很多，因为

  * 最开始的那段if可以跳过
  * adjust_next和remove_next都为假

这次侥幸跳过了复杂的部分，但是躲的了初一，躲不了十五，接下来我们就要面对整个过程最难的部分了。


# vma对物理地址的影响

前面我们讲了很多vma结构本身的操作，如何创建、添加和调整vma，也就是虚拟地址空间的操作。接下来我们要来看看vma这个虚拟地址管家是如何真实管理进程对应的物理地址的。也就是如何影响到page fault的。

要观察这个影响，那免不了要看函数mmap了。

```
    do_mmap()
        addr = get_unmapped_area(file, addr, len, pgoff, flags)
        addr = mmap_region(file, addr, len, vm_flags, pgoff, uf)
            shmem_zero_setup(vma)                                ---  share mem
                file = shmem_kernel_file_setup("/dev/zero")
                vma->vm_file = file
                vma->vm_ops = &shmem_vm_ops
            vma_set_anonymous(vma)                               ---  anonymous mem
                vma->vm_ops = NULL
            call_mmap(file, vma)                                 ---  file back
                file->f_op->mmap(file, vma), setup vma->vm_ops
```

当我们做mmap的时候，会有三种情况，shmem/anonymous/file back。其实仔细说来也就两种区别，设置了vma->vm_ops的和没设置的。

这个设置主要就是影响了page fault时会去调用什么函数去获得真实的物理页。对page fault的代码就不在这里分析了，这里只画一个简图来示意一下mmap和page fault之间的关系：

```
    mmap() calls f_op->mmap()         page fault search vma
       setup vma->vm_ops            call vma->vm_ops->fault()
          |                             to allocate page
          |                                  |
          v                                  |
     file_operations              vma        |
     +-----------------+          +---------------------------+
     |mmap             |          |          |                |
     |                 |--------->|vm_ops    |                |
     +-----------------+          |    +----------------------+
                                  |    |     |                |
                                  |    |     +---> fault      |
                                  |    |                      |
                                  +----+----------------------+
```

mmap设置了对应的vm_ops后，page fault发生时就会找到这个vma，并调用vma->vm_ops来获取真正的物理页。

好了，到这里就到这里。

# 测试程序

最近内核中添加了用户态的测试程序，在这里看下怎么跑的。

```
# cd tools/testing/vma
# make
# ./vma
```

以后有改动就要先通过这个测试了。另外有什么不清楚的，也可以在用户态程序里先跑跑看。
