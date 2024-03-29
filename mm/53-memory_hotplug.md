本小节我们将来看看内存热插拔的一些流程。

# 热插的总体流程

我们先来看一下热插拔的总体流程：

```
add_memory_resource()
	   check_hotplug_memory_range()             --- (1)
     mem_hotplug_begin()
	   memblock_add_node(start, size, nid)      --- (2)
     __try_online_node(nid, start, false)
     arch_add_memory()
		      init_memory_mapping()               --- (3)
		      add_pages() -> __add_pages()        --- (4)
	   create_memory_block_devices()            --- (5)
     __register_one_node()
     link_mem_sections()
          register_mem_sect_under_node()
     firmware_map_add_hotplug()
     mem_hotplug_done()
	   online_memory_block()                    --- (6)
```

其中有几个比较关键的地方解释一下：

* (1) 保证热插拔的内存空间是section对齐
* (2) 把相关的内存信息填写到memblock中
* (3) 填写内核的内存页表
* (4) 又称为hot-add，后面详细说明
* (5) 创建memory_block设备，用户在sysfs上可以看到
* (6) 又称为hot-online，后面详细说明

从上面的流程中可以看到，热插拔的过程包含不少细节。
但是通常来说，主要分成两个部分： hot-add 和 hot-online。

而且这么划分有实际意义。在某些情况下，nvdimm，可以只通过hot-add将内存告知系统，但不添加到buddy system。

# hot-add

hot-add的意义在于给对应的内存分配page struct，也就是有了内存管理的元数据。这样就可以管理内存了。

但此时内存并没有添加到buddy system。恩。。。这是一个值得思考的问题。

好了，还是先来看一下相关的流程：

```
     __add_pages(nid, pfn, nr_pages)
          check_hotplug_memory_addressable()
	        check_pfn_span(pfn, nr_pages, "add")               --- (1)
          sparse_add_section(nid, pfn, pfns, altmap)
                 sparse_index_init(section_nr,  nid)
                 memmap = section_activate(nid, pfn, nr_pages, altmap)
                       memmap = populate_section_memmap(pfn, nr_pages, nid, altmap)
                 page_init_poison()
                 set_section_nid(section_nr, nid)
                 section_mark_present(ms)
                 sparse_init_one_section(ms, section_nr, memmap, ms->usage, 0)
                       ms->section_mem_map |= sparse_encode_mem_map(memmap, pnum)
```

总的来说中规中矩，就是添加了对应的mem_section和memmap，也就是page struct。

有一点值得注意的是在(1)处，这里是保证是subsection对齐的。也就是对于不用hot-online的内存可以是subsection。

# hot-online

hot-online就是要把内存放到buddy system，让内核管理起这部分内存。

```
     online_pages(pfn, nr_pages, online_type)
           zone = zone_for_pfn_range(online_type, nid, pfn, nr_pages)
           move_pfn_range_to_zone(zone, pfn, nr_pages, NULL)
                 init_currently_empty_zone()
                 resize_zone_range(zone, start_pfn, nr_pages)
                 resize_pgdat_range(pgdat, start_pfn, nr_pages)
                 memmap_init_zone()
                       __init_single_page()               --- (1)
                       set_pageblock_migratetype()
           online_pages_range()
                 generic_online_page()
                       __free_pages_core()                --- (2)
                 online_mem_sections()
           shuffle_zone(zone)
           build_all_zonelists(NULL)                      --- (3)
           init_per_zone_wmark_min()
           kswapd_run(nid)
           kcompactd_run(nid)
           writeback_set_ratelimit()
```

几个重要的步骤：

* (1) 初始化page struct
* (2) 将page加入到buddy system
* (3) 重新构造所有NODE_DATA上的zonelist

这么看好像也不是很难。当然魔鬼在细节，仔细看还是有不少内容的。
