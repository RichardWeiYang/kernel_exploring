作为cgroup的一个子系统，自然在初始化时需要遵照cgroup的框架。所以我们先来回顾一下cgroup的框架。

```
cgroup_init_subsys(ss, early)
    ss->root = &cgrp_dfl_root
    css = ss->css_alloc()
    init_and_link_css(css, ss, &cgrp_dfl_root.cgrp)
        css->cgroup = cgrp
        css->ss = ss
    init_css_set.subsys[ss->id] = css
    online_css(css)
        ss->css_online(css)
        css->cgroup->subsys[ss->id] = css
```

其中 ss 是 memory_cgrp_subsys，那么对应调用的函数就是
  * css_alloc  = mem_cgroup_css_alloc
  * css_online = mem_cgroup_css_online

这两个函数除了mem_cgroup_css_online中有一个周期性刷新状态的“工作”，其余做的工作比较直白。

所以初始化的过程没有什么神秘的，但是我们借此机会看一眼memcg的数据结构。mem_cgroup_css_alloc分配的数据结构是mem_cgroup。

这个结构体很长，我把重要的部分分类出来。

```
  mem_cgroup
  +-------------------------------------+
  |memory/swap/memsw/kmem/tcpmem        |
  |    (struct page_counter)            |
  |                                     |
  |thresholds/memsw_thresholds          |
  |    (struct mem_cgroup_thresholds)   |
  |                                     |
  |vmstats                              |
  |    (struct memcg_vmstats)           |
  |vmstats_percpu                       |
  |    (struct memcg_vmstats_percpu)    |
  |                                     |
  |[]nodeinfo                           |
  |    (struct mem_cgroup_per_node*)    |
  +-------------------------------------+
```

对其中的含义做个大致的解释：

  * memory/swap/memsw/kmem/tcpmem 保存了用户设置的限额和当前使用额度
  * thresholds/memsw_thresholds保存了eventfs相关的水线
  * vmstats/vmstats_percpu保存了页的统计数据
  * nodeinf主要和页回收相关

说到这里再提一点，page_counter里自己还保存了一个树形结构。这个结构和cgroup形成的树形结构是一致的。这或许是一个遗留问题。
