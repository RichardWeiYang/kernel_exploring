虚拟内存空间的物理根本可以说就是页表了，没有页表虚拟地址空间是无法幻化出让人眼花缭乱的变化。

# 页表的层次

当然，页表本身已经够让人眼花缭乱了。那就先让我们来看一下页表的样子先。

```
            47               39 38              30 29              21 20              12 11                  0
            +------------------+------------------+------------------+------------------+---------------------+
            |PML4              |Page Directory Ptr|Page Directory    |Page Table        |Offset               |
            +------------------+------------------+------------------+------------------+---------------------+
                   |                    |                      |                     |
                   |                    |                      |                     |
                   |                    |                      |                     |
                   |                    |                      |                     |
  pgd_index(addr)  |    pud_index(addr) |      pmd_index(addr) |     pte_index(addr) |      +----------+
                   |                    |                      |                     |      |          |
                   |                    |                      |                     |      |          |
                   |                    |                      |                     |      +----------+
                   |                    |                      |    pte_offset_map() +----> |pte       |
                   |                    |                      |                            +----------+
                   |                    |                      |     +----------+           |          |
                   |                    |                      |     |          |           |          |
                   |                    |                      |     |          |           |          |
                   |                    |                      |pmdp +----------+           |          |
                   |                    |         pmd_offset() +---->| *pmdp    |---------->+----------+
                   |                    |                            +----------+ pmd_page_vaddr(*pmdp)
                   |                    |                            |          |
                   |                    |                            |          |
                   |                    |                            |          |
                   |                    |     +----------+           |          |
                   |                    |     |          |           |          |
                   |                    |pudp +----------+           |          |
                   |       pud_offset() +---->| *pudp    |---------->+----------+
                   |                          +----------+ pud_pgtable(*pudp)
                   |     +----------+         |          |
                   |     |          |         |          |
                   |pgdp +----------+         |          |
pgd_offset(mm,addr)+---->| *pgdp    |-------->+----------+
                         +----------+
                         |          |
                         |          |
                         |          |
           mm->pgd --->  +----------+

```

其中最上面一行中出现的名词，如PML4，是在intel手册上的。而下方的xx_index/xx_offset是在内核代码中对应使用的名字。

当然上面这个图例已经有点过时了，这个是四层页表的情况，现在已经有五层也表了。

比如在[page table][1]中可以看到五层是这样的。

```
  +-----+
  | PGD |
  +-----+
     |
     |   +-----+
     +-->| P4D |
         +-----+
            |
            |   +-----+
            +-->| PUD |
                +-----+
                   |
                   |   +-----+
                   +-->| PMD |
                       +-----+
                          |
                          |   +-----+
                          +-->| PTE |
                              +-----+
```

## 页表层级常用helper

首先我们从上面图中看到内核中对页表每一层都起了自己的名字：

  * pgd
  * p4d
  * pud
  * pmd
  * pte

在操作对应层级时，也有对应的helper帮助我们获取对应的信息。先按照功能我来分个类：

  * 遍历型：用于遍历页表
  * 访问型：用于访问页表项内容
  * 分配型：用于分配页表

接下来就按照这几个大类来看看内核中常用的helper。 其中xxx代表了 pgd/pud/pmd/pte。

遍历型：

  * xxx_index(address):            获取对应层级偏移量，用来计算下级页表地址
  * xxx_offset(xxx_t *, addr):     第一个参数指向的页表起始地址 + xxx_index()，也就是往下一层级页表走一层。比如pmd_offset()，传入参数是pud_t *，得到的是pmd_t *。
  * pte_offset_map(pmd_t *, addr): 没有pte_offset(), 还有一个pte_offset_kernel(pmd_t *, addr)

其中xxx_offset()值的注意的是，除了pgd_offset()，其余变体都是从上一层级的页表项中获取下一层级的页表虚拟地址，然后加上xxx_index()得到的。

另外，从含义上来说pte_offset_kernel()和其他的xxx_offset是一样的。pte_offset_map()是在pte_offset_kernel()上又做了一些数据校验。

访问型：

  * xxxp_get(xxx_t *):           获得当前页表项(xxx_t *)的内容，读出指针指向的地址里的内容。用作下面一类helper的入参。
  * xxx_val(xxx_t ):             获取xxx_t对应的值。注意这个和xxxp_get()的区别。xxx_val()才会真正去读出xxx_t这个类型中的值。
  * xxx_flags(xxx_t ):           在xxx_val()的基础上，取出页表项相关的属性位
  * xxx_none(xxx_t ):            判断xxx_t对应这个entry是否为空，空说明需要分配下级页表了
  * xxx_present(xxx_t ):         判断xxx_t对应这个entry是否存在，其实是看下一层页表是否存在
  * xxx_pfn(xxx_t ):             获取页表项指向的页的pfn，在xxx_val()基础上去掉不相关的bit，再右移PAGE_SHIFT
  * xxx_page(xxx_t ):            获取页表项指向的页的page结构, 将xxx_pfn()转换为page struct
  * pmd_page_vaddr(xxx_t ):      获取页表项指向的页的虚拟地址。PS:这个和获取xxx_page()的过程很像，前者是拿到pfn后转换为page struct，后者是将pfn转换为虚拟地址。
  * xxx_pgtable(xxx_t ):         部分有定义，实际就是xxx_pfn()转换成虚拟地址，再做一个类型转换。

其中只有xxxp_get()的入参是指针，其余都不是。通常理解xxxp_get()的返回值，会用作后续的入参。但实际使用中常常见到pgd_none(*pgd)这样的情况。

其中pmd_pgtable()是个特例，在大多数平台下他默认定义为pmd_page()。平台可以在asm/pgtable.h中覆盖这个定义。难怪单独有一个pmd_page_vaddr的定义。

分配型：

  * xxx_alloc():                 如果已经有页表，返回结果同xxx_offset()；否则分配xxx对应层级的页表
  * xxx_populate():              安装页表，把下一层新分配的页表地址填到xxx表示的这一层。这是xxx_alloc()中的一部分
  * xxx_install():               和xxx_populate()差不多，多了一个判断，最后调用xxx_populate()

## 图示

个人觉得，内核中这些helper写得不是很统一，有些含义不是特别清楚，容易混淆。用图的形式可能更容易理解。

```
                            PMD
   pud_pgtable(pudp) / ---->+------------------+<---- pmd_pgtable_page(pmdp)
     pud_val(pud)        ^  |                  |         pmd_pgtable_page()返回的是pmdp
                         |  |                  |              所在的PMD这个页的struct page
                         |  |                  |
        pmd_index(addr)  |  |                  |
                         |  |                  |
                         v  +------------------+                        PTE
   pmdp                 --->|                  | pmd_page_vaddr(pmdp)-->+------------------+<--- pmd_page(pmd)
    = pmd_offset(pudp, addr)+------------------+   pmd_val(pmd)      ^  |                  |        pmd_page()返回的是
                            |                  |                     |  |                  |           pmdp中保存的页表的struct page
                            +------------------+                     |  |                  |
                                                    pte_index(addr)  |  |                  |
   pud_pgtable() / pmd_offset() 返回的是虚拟地址                     |  |                  |
   pud_val() 返回的是物理地址                                        v  +------------------+
                                                 ptep                -->|                  |
                                                  = pte_offset_kernel() +------------------+
                                                                        |                  |
                                                                        +------------------+

                                                 pmd_page_vaddr() / pte_offset_kernel()
                                                     返回的是虚拟地址
                                                 pmd_val()返回的是物理地址

```

# 页表的填写

那这张表怎么填写呢？当然途径不止一条，不过最重要的就是**缺页中断**了。

总的来讲就是按照虚拟地址来遍历整个页表，根据不同PTE的状态做不同的处理。

## 缺页中断

首先是架构相关的中断处理程序代码：

```
exc_page_fault
  handle_page_fault
    do_kern_addr_fault
    do_user_addr_fault
      vma = lock_vma_under_rcu()              <--- 锁住对应vma
      handle_mm_fault(FAULT_FLAG_VMA_LOCK)    <--- 架构无关代码
      vma_end_read()

      ... or

      vma = lock_mm_and_find_vma()            <--- 锁住mmap_lock
      handle_mm_fault()                       <--- 架构无关代码
      mmap_read_unlock()
```

## 架构无关代码

然后就是架构无关的缺页处理代码：

```
handle_mm_fault(vma, address, flags, regs)
    if (is_vm_hugetlb_page(vma))
        hugetlb_fault()                <--- hugetlb处理
    else
        __handle_mm_fault
            // pud/pmd level
            // pte level
            handle_pte_fault           <--- 包括PTE这层页表，和做后的page
                // empty pte
                do_pte_missing
                    do_anonymous_page
                    do_fault
                // other1
                do_swap_page
                do_numa_page
                // other2
                do_wp_page
                pte_mkdirty
                pte_mkyoung
```

## 匿名页填写

页表按照映射对象来分主要是两种：

 * 匿名页表
 * 文件页表

这里我们先看匿名页表 -- do_anonymous_page。

```
do_anonymous_page()
  pte_alloc()                        <--- 这里会分配pte这一层页表，如果没有的话
  // 如果不是写，用zero-page
  entry = pte_mkspecial(pfn_pte(my_zero_pfn(), ..));

  // 准备匿名映射
  ret = vmf_anon_prepare()
    ret = __vmf_anon_prepare(vmf)
      __anon_vma_prepare(vma)        <--- 分配或者查找相邻可用vma->anon_vma
    return ret
  // 准备真正需要映射的内存
  folio = alloc_anon_folio(vmf);
    // THP or not
    folio_prealloc(, vma, vmf->address, true)
      vma_alloc_folio(GFP_HIGHUSER_MOVABLE, )    <-- 指定了可用的zone

  // 最后设置到pte页表中
  set_ptes()
```

# 页表的释放

> 有借必有还，有写必有擦。

看过了页表构造的过程（虽然糙了点），那也该看看页表释放的过程。当然我不确定是不是有很多地方可以做释放的动作，不过下面这个函数是我找到的接口之一。

```
unmap_region
    unmap_vmas()
    free_pgtables()
```

其中unmap_vmas释放了真正对应的内存，而free_pgtables才释放页表。

写到这里，感觉写完了，估计是很多细节自己还不知道。没事，留着以后有新发现再来挖掘。

# 页表上的锁

页表是一个公共资源，当发生缺页中断时大家可能同时访问页表并进行操作。

最开始的时候，每个进程只有一把大锁，mm->page_table_lock。

为了序列化对页表的访问，内核中提供了各种层次的锁来保护。 是的，就是我们在前面看到的页表层级，内核为不同的层级定义了不同的锁。

  * pud_lockptr(mm, pudp)
  * pmd_lockptr(mm, pmdp)
  * pte_lockptr(mm, pmdp)
  * ptep_lockptr(mm, ptep)

目前pud_lockptr()是个冒牌货，因为还没有发现有扩展性的问题。

其他的锁搜保存在特定的页表页中

  * pmd_lockptr(): pmd_pgtable_page(pmd)
  * pte_lockptr(): pmd_page(*pmdp)
  * ptep_lockptr(): virt_to_page(ptep)


直观一些，我慢来看看这几个锁在那里。

```
      PMD               +---pmd_lockptr()
      +--------------+<-+                    /--pte_lockptr()
      |              |        PTE           v   ptep_lockptr()
      |              | -----> +--------------+
      |              |        |              |
      +--------------+        |              |
                              |              |
                              +--------------+
```

也就是pmd_lockptr()存放在PMD的页表里，而pte_lockptr()和ptep_lockptr()都在PTE页表里。

# 参考文档

[内核文档 -- page tables][1]

[1]: https://docs.kernel.org/mm/page_tables.html
