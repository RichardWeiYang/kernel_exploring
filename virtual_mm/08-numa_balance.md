在上一节中我们对内存分配策略做了一定的了解，并且留了一个话题 -- numa balance。也就是内核为了在进程迁移过程中保持内存分配策略做的工作。

进程在运行过程中有可能会迁移到不同的cpu上执行，如果被迁移到的cpu和内存之前不满足mempolicy，那么numa balance就会尝试将用符合策略的内存来替换。所以整个过程分为两个步骤：

  * 扫描内存是否符合策略，并将不符合的pte设置为pronnone
  * 通过numa hinting page fault替换页

# 扫描进程空间

说到扫描就有一个扫描时机，当使用cfs调度时，这个时机由task_tick_fair时刻触发。经过一系列的检查后，会插入curr->numa_work等待执行。

这个numa_work的回调函数是task_numa_work，而这个函数的核心就是扫描符合条件的vma，并通过change_prot_numa把对应的pte属性添加上PAGE_NONE。

# numa hinting page fault
