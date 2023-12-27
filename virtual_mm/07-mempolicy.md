随着系统中的CPU和内存增加，物理上将内存分成了numa节点。而运行时系统从哪个节点上分配内存就会影响到系统运行的性能。

内核当然可以提供一种默认的分配策略，而同时用户或许也会希望能够设置自己的分配行为。

今天我们就来看看内核提供了哪些分配策略，又是如何实现的。

# 最关键的数据结构 -- mempolicy

对我来说，每次要学习一个新的概念，最关键的就是学习这个概念背后的数据结构，而算法则是对数据结构的一个操作。

在numa策略中，内核最关键的数据结构是mempolicy。

```
    mempolicy
    +------------------------------+
    |refcnt                        |
    |    (atomic_t)                |
    |mode                          |
    |flags                         |
    |    (unsigned short)          |
    |v                             |
    |   +--------------------------+
    |   |preferred_node            |
    |   |   (short)                |
    |   |nodes                     |
    |   |   (nodemask_t)           |
    |   +--------------------------+
    |w                             |
    |   +--------------------------+
    |   |cpuset_mems_allowed       |
    |   |user_nodemask             |
    |   |   (nodemask_t)           |
    +---+--------------------------+
```

这个结构相对来说是简单的。其中最重要的就是mode成员：

  * mode 指定了策略的模式

知道了这个策略的模式，你就大致能猜到策略的运作和其他成员的含义。那我们就来看看内核当前都有哪些模式：

```
    enum {
    	MPOL_DEFAULT,
    	MPOL_PREFERRED,
    	MPOL_BIND,
    	MPOL_INTERLEAVE,
    	MPOL_LOCAL,
    	MPOL_MAX,	/* always last member of enum */
    };
```

具体每个策略的含义暂且不表，因为这个可以在网上直接搜到。而我们想要说的是，这个数据结构是如何影响到一个进程的内存分配的。

# mempolicy和进程的关联

既然是要控制进程的内存分配策略，那么必然这个数据结构就要和进程发生关系，否则怎么能够控制到进程呢？

这时候，还是数据结构能帮上忙。

```
                 task_struct
                 +----------------------+
                 |mempolicy             |
                 |   (struct mempolicy*)|
                 |mm                    |
                 |   (struct mm_struct*)|
                 +----------------------+
                    /               \
                 /                    \
              /                         \
           /                              \
      vma                              vma
      +------------------------+       +------------------------+
      |vm_ops                  |       |vm_ops                  |
      |    get_policy          |       |    get_policy          |
      |                        |       |                        |
      |vm_policy               |       |vm_policy               |
      |    (struct mempolicy*) |       |    (struct mempolicy*) |
      +------------------------+       +------------------------+
```

从这个结构中我们可以看到，在进程和vma级别都有各自的mempolicy指针。当我们在分配内存时，就会根据对应的mempolicy来分配。

那接下来我们就要回答两个问题：

  * 如何设置mempolicy
  * mempolicy如何影响分配

# 设置mempolicy的用户态接口

首先我们来回答第一个问题--如何设置mempolicy。因为mempolicy分别在进程和vma两个层次，所以内核提供了对应的两个接口来设置。

  * set_mempolicy
  * mbind

## 设置进程级numa策略 -- set_mempolicy

首先是进程级别的numa策略设置，由函数set_mempolicy来负责。

具体函数的含义大家查找man就可以了，我们这里来看看这个函数干的活。

```
    set_mempolicy() -> kernel_set_mempolicy()
        get_nodes(&nodes, nmask, maxnode)
        do_set_mempolicy(mode, flags, &nodes)
            new = mpol_new(mode, flags, nodes)
            task_lock(current)
            mpol_set_nodemask(new, nodes, scratch)
            old = current->mempolicy
            current->mempolicy = new
            task_unlock(current)
            mpol_put(old)
```

打开一看其实很简单，就是给task_struct上新安装一个mempolicy。得嘞。

## 设置区域级numa策略 -- mbind

除了进程级别的numa策略，内核还提供了vma级别细粒度的策略设置，由函数mbind来负责。

mbind函数除了设置vma级别的numa策略，还会做内存迁移。所以这个函数实际上是做了两件事情。

```
    mbind(start, len, mode, nmask, maxnode, flags) -> kernel_mbind()
        get_nodes(&nodes, nmask, maxnode)
        do_mbind(start, len, mode, mode_flags, &nodes, flags)
            new = mpol_new(mode, mode_flags, nmask)
            migrate_prep()
                lru_add_drain_all()
            queue_pages_range(mm, start, end, nmask, flags | MPOL_MF_INVERT, &pagelist)
            mbind_range(mm, start, end, new)
            migrage_pages(&pagelist, new_page, NULL, start, )
```

所以整个函数只有在mbind_range上才是去更新vma对应的策略，其他的步骤都是为了做内存迁移准备的。

# numa策略的作用

了解了策略的数据结构，了解了这个结构和进程之间的关系，也了解了如何设置策略，那么接下来就是要看看这个设置好的策略究竟如何起作用了。

大家猜猜，内核中会在哪些地方起作用呢？嗯，对了，至少有两个地方会发挥numa策略的作用。

  * page fault
  * numa balance

首先是在缺页处理时当然会按照当前的策略去分配内存，其次就是会动态的检测进程的内存是否符合当前的策略并进行相应的调整。

## page fault

发生缺页异常时，内核会去分配内存并将页表填好。此时就会考虑到从哪里去分配内存，在这个时候就会去参考之前设置的numa策略。

```
do_anonymous_page()
    alloc_zeroed_user_highpage_movable() -> __alloc_zeroed_user_highpage()
        alloc_pages_vma(), alloc with NUMA policy(understand how policy works)
            pol = get_vma_policy(vma, addr), pol from vma or task policy
            alloc_page_interleave(), if pol->mode == MPOL_INTERLEAVE
            nmask = policy_nodemask()
```

比如说在我们熟悉的do_anonymous_page()过程中需要去分配页，就会调用alloc_pages_vma()根据对应的mempolicy来分配。

## numa balance

而另一个重要的场景是numa balance，这个部分因为确实有点长，我们需要另开一节来详细描述。
