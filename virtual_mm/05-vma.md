既然每个进程有自己的虚拟地址空间，那么就需要有东西把它管理起来。在内核中，这个数据结构就是vm_area_struct，简称vma。

# 两种结构的双生儿

vma是一个非常有意思的数据结构，它融合了两种常见的数据结构：

  * 红黑树
  * 双链表

当然这个vma的目的是用来管理整个进程的虚拟地址空间。也就是当我们需要在虚拟地址空间上查找，分配一段的时候，就会使用到vma结构。

# 单个vma的内容

虽然一叶障目是不太好的，但是为了了解整个vma树，我们还是得从vma结构体这片叶子说起。

下面是我简化后的一个vma结构体的重要成员。

```
    vm_area_struct
    +--------------------------------+
    |vm_rb                           |    vma rb tree node
    |   (struct rb_node)             |
    |vm_prev, vm_next                |    vma list in order
    |   (struct vm_area_struct*)     |    * no overlap to each other
    |                                |
    +--------------------------------+
    |vm_start, vm_end                |    the range we cover
    |   (unsigned long)              |
    |                                |
    |vm_file                         |
    |   (struct file*)               |
    |vm_pgoff                        |    offset in PAGE_SIZE
    |   (unsigned long)              |
    +--------------------------------+
    |rb_subtree_gap                  |    http://tinylab.org/rbtree-part1/
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

  * 最上面是树和链表的架子，每一个vma结构体由他们搭建成树和链表
  * 接下来是当前vma在虚拟空间中所占的位置，以及对应的文件及在文件中的位置
  * 然后这个很有意思，这是对红黑树的一个优化，为了加速判断子树中是否有足够的空间
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
  * find_vma_link(mm, addr, end, ...)

但是用途上略有差别，前者是只为了查找来的，而后者是为了插入做准备的。

而且find_vma函数和我们普通的查找还不一样，我们一般想象的是这个函数会返回一个vma，这个vma是覆盖地址addr的。但是你仔细看这个函数，其实返回的vma表示的是第一个满足addr < vm_end条件的。也就是这个addr可能并不在任何vma空间内。

而find_vma_links这个函数就有意思了。如果vma只是一个链表，那么我只要找到其中某一个节点就知道应该插入到哪里。但是vma还是一个红黑树，所以在插入节点时需要知道父节点和究竟是父节点的哪个孩子。所以find_vma_links看上去会复杂一些。另外该函数还会判断是否有重叠，如果发生重叠就返回错误。这一点说明了整个vma树上，所有节点的范围都是分开的。

仔细看这个函数，你会觉得这个函数写得可真是优美。

## 插入/删除

插入和删除可以说是天生的一对，有插入那就有删除。不过这老天爷规定的事儿在有了区块链之后就被打破了，硬生生只留"插入"在人间。

好了，还是不提这伤心事儿了，说说vma的插入和删除吧：

  * 插入__vma_link
  * 删除__vma_unlink_common

其中所做的事情也无非就是链表和树的增删了。值得注意的一点是增删的时候都会做vma_gap_update。

还记得在vma结构体中有一个成员叫rb_subtree_gap吗？对了就是更新它的。这个值记录了以当前vma节点为根节点子树中最长的空间范围。这样下次再查找可用空间是就可以先判断这个值，再去搜索了。这个工作在函数unmapped_area/unmapped_area_topdown这两个函数中进行。

随着代码阅读的深入，突然发现事情不妙。因为发现了插入/删除这件事上还出现了“第三者”--detach_vmas_to_be_unmapped。

这个函数是用来批量删除的，干的事情其实和单个删除差不多，只是将要删除的vma们打包删除了。

## 拆分/整合

拆分和整合在内核中主要有两个大的函数来处理：

  * split_vma
  * vma_merge

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

### vma_merge

vma_merge这个函数可以说是我见过的比较复杂的函数之一了。还好内核开发者比较友好，在函数头上给大家列出了vma_merge会处理的几种情况。

```
     AAAA             AAAA                   AAAA
    PPPPPPNNNNNN    PPPPPPNNNNNN       PPPPPPNNNNNN
    cannot merge    might become       might become
                    PPNNNNNNNNNN       PPPPPPPPPPNN
    mmap, brk or    case 4 below       case 5 below
    mremap move:
                        AAAA               AAAA
                    PPPP    NNNN       PPPPNNNNXXXX
                    might become       might become
                    PPPPPPPPPPPP 1 or  PPPPPPPPPPPP 6 or
                    PPPPPPPPNNNN 2 or  PPPPPPPPXXXX 7 or
                    PPPPNNNNNNNN 3     PPPPXXXXXXXX 8
```

说实话这个函数要结合vma_adjust实在是太难了。我看了不下十遍吧，也不敢说完全看懂了。更别提要我去从头实现，估计根本想不到从哪里入手。

vma_merge函数本身逻辑还是很清晰的，尤其是结合上边的注释来看的话。这个函数所做的事情就是判断新的区域是否可以和前面的合并？是否可以和后面合并？然后根据不同的情况调用vma_adjust去做调整。

接下来就是神奇的vma_adjust中对应的部分了。从代码实现上看，作者的逻辑是按照end的取值范围来做划分的：

```
          vma                 next
      +-----------+        +------------+
            ^                    ^              ^
            |                    |              |
         case 4               case 5         case 1,6,7,8
```

这么一分析，我突然明白了。作者是将vma_merge中六个需要调整next的情况按照end的位置分成了三种情况。

  * end大于next
  * end在next内
  * end在vma内

而case 2,3 因为不需要调整next。或者更加准确的讲是只需要调整一个vma的状态，所以不在vma_adjust函数中最开始的那个if条件中处理。

接着我们再打开需要调整next的六种情况，其中又可以分成两种

  * 删除next, case 1,6,7,8
  * 调节next, case 4,5

到这里我们把上述的分析总结一下，看看vma_merge的八中情况到了vma_adjust都有哪些分类：

```
 +-- 不动 *next* -- case 2, 3
 |
 |
-+                +-- 删除 *next* -- case 1,6,7,8
 |                |
 |                |
 +-- 调整 *next  --+
                  |
                  |
                  +-- 调整 *next* -- case 4,5
```

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
