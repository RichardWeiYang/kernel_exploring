使用memcg最常用的目的就是限制内存使用。这一节我们就来看看用户如何设置内存使用限制，下一节我们就来看看内核是如何做到对内存记账的。

# memory.limit_in_bytes

在[使用cgroup控制进程cpu和内存][1]中我们已经看到，用户可以通过写memory.limit_in_bytes来限制memcg的大小。这里我们就来看看这个具体过程。

```
mem_cgroup_write()
    mem_cgroup_resize_max(memcg, nr_pages, false)
        counter = &memcg->memory
        page_counter_set_max(counter, max)
        try_to_free_mem_cgroup_pages(memcg, 1, GFP_KERNEL, !memsw)
```

设置内存大小限制的逻辑还是比较简单的。用一句话来概括就是设置memcg->memory这个page_counter的max值。

如果设置最大值失败，也就是当前使用已经超过我们要设置的值，那么就会调用try_to_free_mem_cgroup_pages尝试直接回收一些内存，然后再试。

这部分确实挺简单的，恭喜你又掌握了一个知识点。

[1]: /cgroup/01-control_cpu_mem_by_cgroup.md
