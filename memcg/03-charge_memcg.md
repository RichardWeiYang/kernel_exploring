既然是要限制内存使用，自然需要一个办法来对内存使用记账。也就是将内存的消耗对应到memcg，这样才有可能知道内存是否超过限制并采取行动。

# 两个入口

经过调研，当前内核5.17一共有两个对外的入口做memcg记账

  * mem_cgroup_charge
  * mem_cgroup_swapin_charge_page

前者是在做page fault时，而后者是在swapin时。别看就这两个入口，但在内核里要找全安插这两个入口的点可真是不容易。没有对内核全面的了解，真是做不到。所以在现在内核的框架下，能统一这个入口是不是也是一个改进方向呢？

# 核心函数 charge_memcg

不论是从哪个入口对memcg做记账，都会调用到函数charge_memcg()。

```
  charge_memcg(folio, memcg, gfp)
      try_charge(memcg, gfp, nr_pages)
          page_counter_try_charge(&memcg->memsw, batch, &counter)
          page_counter_try_charge(&memcg->memory, batch, &counter)
          mem_over_limit = mem_cgroup_from_counter(counter, memory);
          memcg_memory_event(mem_over_limit, MEMCG_MAX)
          try_to_free_mem_cgroup_pages(mem_over_limit, )
          mem_cgroup_oom(mem_over_limit, gfp)
      css_get(memcg->css)
      commit_charge(folio, memcg)
          folio->memcg_data = memcg
      mem_cgroup_charge_statistics(memcg, nr_pages)
      memcg_check_events(memcg, folio_nid(folio))
```

简单来说就是尝试用page_counter_try_charge()做记账，如果没成功就用try_to_free_mem_cgroup_pages()或者mem_cgroup_oom()回收一些资源然后再尝试。
