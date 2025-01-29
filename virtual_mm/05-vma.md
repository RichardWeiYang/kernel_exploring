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

查找的函数

  * find_vma(mm, addr)

find_vma函数和我们普通的查找还不一样，我们一般想象的是这个函数会返回一个vma，这个vma是覆盖地址addr的。但是你仔细看这个函数，其实返回的vma表示的是第一个满足addr < vm_end条件的。也就是这个addr可能并不在任何vma空间内。

## 合并

  * vma_merge_new_range()
  * vma_merge_existing_range()

### vma_merge_new_range

当映射一个新的空间，即这段空间在当前虚拟地址空间中为空，会尝试这个函数。

看看是否能和当前相邻的空间合并。

### vma_merge_existing_range

这个是已有的一个空间，发生了属性变化后，看看是否能和相邻的空间合并。

## 拆分

  * split_vma

### split_vma

因为某些原因要把当前已有的某个空间拆成两个。

## 修改

  * vma_modify()

通常我们要修改某一段地址空间属性时会用到这个。

# 迭代器

当vma脱离了红黑树，使用maple tree后，对内存映射树的访问就引入了迭代器。因为原先的双生结构，vma自带了位置信息。而脱离后，需要迭代器记录位置，方便前后搜索，不用从头再来。

## vma_find(vmi, max)

vma_find()一般作为迭代器刚开始第一次调用，作用是查找[vmi.index, max]之间第一个非NULL值。

## vma_next(vmi)

和vma_find()一样，只是这次查找的范围是[vmi.index, ULONG_MAX]。

值得注意的是，如果vmi.index这个位置有值，则会被返回。而不是通常理解的，会跳过vmi.index去找到下一个。

## vma_prev(vmi)

向前查找，范围是[0, vmi.index]。

这个和我们平时理解的一样，是真的去找到前面一个。也就是如果vmi.index上有值，是会被跳过的。

## vma_iter_next_range(vmi)

查找下一个范围，不论这个范围有没有值都返回。查找范围是[0, vmi.index]

这个也和我们平时理解的一样，是真的去找后面一个。

## vma_iter_next_rewind(vmi, **prev)

这个比较神奇，是上面三个函数的组合。

其目的是为了找到当前范围，前后有值的那两个值。然后再把vmi的范围恢复到当前的范围。

# 锁机制

进程地址空间的访问（缺页中断）和改变（mmap/mprotect）是系统核心的操作，而缺页中断则是非常频繁的。
为了优化这部分的性能，同时又避免地址空间损坏，内核中提供了非常精巧的锁机制。

目前内核引入了Per-VMA lock，具体的commit为

```
commit 0b6cc04f3db3604c1485049bc9582523c2b44b75
Author: Suren Baghdasaryan <surenb@google.com>
Date:   Mon Feb 27 09:36:08 2023 -0800

    mm: introduce CONFIG_PER_VMA_LOCK
```

## 内核文档

首先内核中有一个文档详细讲解这部分的内容。

Documentation/mm/process_addrs.rst

为了方便看也可以生成html文档，用浏览器阅读。具体方法可以看[内核自带文档][1]，如何生成文档。

PS: 另外LWN也有关于[Per-VMA lock][2]的解释，以及[稳定下来][3]的情况。

## 图解

这里我再用自己的理解解释一遍：

```
                                      vma
                                      +--------------+
   mm                                 |vm_lock       |
   +--------------+                   |vm_lock_seq   |
   |mmap_lock     |                   +--------------+
   |mm_lock_seq   |
   +--------------+                   vma
                                      +--------------+
                                      |vm_lock       |
                                      |vm_lock_seq   |
                                      +--------------+

                                      vma
                                      +--------------+
                                      |vm_lock       |
                                      |vm_lock_seq   |
                                      +--------------+
```

先来看看实现锁机制相关的数据结构。

  * mm_struct： 一个读写信号量，一个顺序值
  * vma：       一个读写信号量，一个顺序值

可以看出，在mm_struct和vma中有着相似的数据结构支撑锁机制。

以前的版本中，只有mmap_lock这个大锁，所有访问/修改地址空间的操作都要获取这一个大锁。这样就产生了瓶颈。
现在引入一个顺序值，通过他减少在读地址空间时对大锁的竞争。

先看下读的过程：

```
开始读：
    down_read(vm_lock)
结束读：
    up_read(vm_lock)
```

再看下写的过程：

```
开始写：
    down_write(vm_lock)
      vm_lock_seq <- mm_lock_seq
    up_write(vm_lock)
结束写：
    mm_lock_seq <- mm_lock_seq + 1
```

这样，当访问vma时，我们就可以只锁vm_lock达到目的，减少了多mmap_lock的竞争。

注意一个细节，读的保护范围是从读开始一直到读结束，而写的保护范围只是同步顺序值的时候。因为这里隐藏了每次写都要拿到mmap_lock的前提。
恩，这个也是蛮有意思的。

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

# 进程地址空间布局

虽然我们可以通过mmap分配地址空间给进程，但是也不是我们随意想分哪里就分哪里的。

比如x86架构下，具体空间如何使用可以看[mm layout][5]

# 功能测试

最近内核中添加了用户态的测试程序，在这里看下怎么跑的。

```
# cd tools/testing/vma
# make
# ./vma
```

以后有改动就要先通过这个测试了。另外有什么不清楚的，也可以在用户态程序里先跑跑看。

# 性能测试

社区里不少开发者在改动完mm相关代码后会做一个性能测试。目前常看到的是[will-it-scale][4]

## 如何运行测试

这个github的文档毕竟是开发者写的，还真的不是那么友好。

其实用起来不难，但是没点介绍还真开始不了。

首先要make， 这样才能有实际的测试程序。

PS: 有可能缺少hwloc库。可以通过apt install libhwloc-dev安装。

运行的话有几种：

  * 全部运行
  * 运行单个测试用例
  * 运行单个用例中进程或线程

全部运行的话比较暴力

```
./runalltests
```

但这么多太耗时了，也不一定是你想要的测试。

打开runalltests这个脚本，实际就是挨个调用了runtest.py。传入的参数是测试的名字，比如

```
./runtest.py mmap1 > mmap1.csv
```

这样可以单独执行mmap1这个测试用例。

再打开这个python脚本，可以看到脚本最终运行的是：

  * mmap1_process
  * mmap1_threads

这两个可执行程序，所以如果你想更直接点，那就直接运行这两个可执行文件把。

一般执行会加参数，可以参照runtest.py脚本中的参数，也可以运行./mmap1_process -h获得运行帮助。

## 测试框架解析

这个测试框架很有意思

  * 每一个可执行文件都是由main.c和tests目录下的一个c文件一起编译出来的
  * 每个tests目录下的一个c文件都会生成两个可执行文件，一个进程版本和一个线程版本
  * runalltests就是把所有生成出来的测试用例都执行一遍runtest.py
  * 而runtest.py是把一个case中的进程版和线程版都跑一遍
  * runtest.py设置了每次运行时的参数。 -s 5执行5次，这个不变。-t 用多少进程和线程是一个等差数列，从1到nr_cores。

现在我们再打开main.c看一下。

  * 准备了results[]的一段共享空间，每个进程/线程会把结果写到这里
  * new_task_affinity()会启动进程/线程，最后运行的函数是testcase
  * 而这个testcase函数是在每个tests目录下c文件中定义的
  * 接下来main函数每隔1秒中读取results[]中的值，并得出最大最小，最后算出平均值
  * 到指定的执行次数后，杀掉所有进程/线程

而在具体的测试用例中，代码就很简单了。

  * 一个无限循环，每次循环增加results[]的计数器

所以整个测试框架就是**在指定次数，也就是指定多少秒内，某个操作可执行多少次**。

通过这个来比较性能上的变化。

## 图示化结果

用runtest.py运行后会生成.csv文件，在这个基础上再运行postprocess.py可以生成一个html，然后可以在浏览器中查看图形的数据。

PS：要在当前目录上打开，因为依赖plot.js这个文件。

比如运行mmap1测试后，生成的图是这样。

![mmap1](/virtual_mm/will-it-scale.png)

先看这张图的坐标系：

  * 横坐标是运行测试时的进程数或者是线程数
  * 左边的纵坐标是每秒钟的平均执行次数
  * 右边的纵坐标是系统idle的百分比

再来看图中的几条曲线：

  * 蓝色/黄色分别是进程/线程在不同数量下执行次数
  * 绿色的点 表示理想线性情况
  * 红色/紫色分别是进程/线程在不同数量下系统空闲百分比

所以从这张图上看，线程比进程更接近线性理想情况，但两者情况都不太好。



[1]: /reference/03-kernel_doc.md
[2]: https://lwn.net/Articles/906852/
[3]: https://lwn.net/Articles/937943
[4]: https://github.com/antonblanchard/will-it-scale
[5]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/arch/x86/x86_64/mm.rst
