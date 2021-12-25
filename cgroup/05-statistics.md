既然要管理系统的资源使用，那就需要有方法来统计资源使用。为此cgroup在当前内核版本(v5.16)，采用了rstat的结构来统计。

# rstat结构

在每个基础cgroup中有一组结构体用来记录状态变化，这组结构体统称为rstat。这组结构体长这个样子：

```
    +======================================+
    |bstat                                 |
    |last_bstat                            |
    |    (struct cgroup_base_stat)         |
    |                                      |
    |rstat_cpu                             |
    |    (struct cgroup_rstat_cpu* _percpu)|
    |    +---------------------------------+
    |    |updated_children                 |
    |    |updated_next                     |
    |    |    (struct cgroup*)             |
    |    |                                 |
    |    |bsync                            |
    |    |    (struct u64_stats_sync)      |
    |    |     +---------------------------+
    |    |     |seq                        |
    |    |     |    (seqcount_t)           |
    |    |     +---------------------------+
    |    |bstat                            |
    |    |last_bstat                       |
    |    |     (struct cgroup_base_stat)   |
    |    |     +---------------------------+
    |    |     |cputime                    |
    |    |     |    (struct task_cputime)  |
    |    |     |    +----------------------+
    |    |     |    |stime                 |
    |    |     |    |utime                 |
    |    |     |    |    (u64)             |
    |    |     |    |sum_exec_runtime      |
    |    |     |    |    (unsigned long)   |
    |    |     +----+----------------------+
    |    +---------------------------------+
    |                                      |
    +======================================+
```

其中最主要的有两部分：

  * cgroup_base_stat 用来记录数据
  * updated_children/updated_next 用来记录谁需要更新

# 如何更新

当整个cgroup树形结构增加时，如何避免不必要的统计动作，成了系统优化需要考虑的问题。对rstat来讲，updated_children/updated_next就是起到这个作用的。

参与这个工作主要有两个函数：

  * cgroup_rstat_updated
  * cgroup_rstat_cpu_pop_updated

第一个是添加，第二个是删除。

话不多说，咱来看看效果图。

假设最开始cgrp1并没有数据改变

```
                                  cgroup_root
                                  +------------------+<------------------------------+
                                  |                  |                               |
                                  |updated_children--|---+                           |
                                  +------------------+   |                           |
                                                         |                           |
                               +-------------------------+                           |
                               |                                                     |
                               |                                                     |
       cgrp1                   |  cgrp2                      cgrp3                   |
       +------------------+    +->+------------------+<----+ +------------------+    |
       |updated_next      |       |updated_next   ---|------>|updated_next   ---|----+
       |updated_children  |       |updated_children--|--+  | |updated_children  |
       +------------------+       +------------------+  |  | +------------------+
                                                        |  |
                               +------------------------+  |
                               |                           |
                               |  cgrp2'                   |
                               +->+------------------+     |
                                  |updated_next    --|-----+
                                  |updated_children  |
                                  +------------------+
```

cgrp1中数据有改变之后,cgrp1也会加入到这个链表中。

```
                                  cgroup_root
                                  +------------------+<------------------------------+
                                  |                  |                               |
                                  |updated_children--|---+                           |
                                  +------------------+   |                           |
                                                         |                           |
    +----------------------------------------------------+                           |
    |                                                                                |
    |                                                                                |
    |  cgrp1                      cgrp2                      cgrp3                   |
    +->+------------------+       +------------------+<----+ +------------------+    |
       |updated_next   ---|------>|updated_next   ---|------>|updated_next   ---|----+
       |updated_children  |       |updated_children--|--+  | |updated_children  |
       +------------------+       +------------------+  |  | +------------------+
                                                        |  |
                               +------------------------+  |
                               |                           |
                               |  cgrp2'                   |
                               +->+------------------+     |
                                  |updated_next    --|-----+
                                  |updated_children  |
                                  +------------------+
```

所以说白了, updated_children/updated_next还是一种树形结构。其中只把有状态更新和需要更新的节点包含了进来。

# 更新顺序

多说一句，cgroup_rstat_cpu_pop_updated这个函数还是挺有意思的。因为cgroup的树形结构的要求，子节点的状态需要汇总到父节点，所以更新的时候就要求从子节点更新，这样才能让父节点得到正确的统计。
