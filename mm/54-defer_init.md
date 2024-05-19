defer_init，内核中的延迟满足

在上一节的内容中[page结构体，我要如何安放你的灵魂][1]中我们看到了为了应对海量内存，内核是如何安放page结构体的。然而到了这里，事情还没有结束。紧接着应对海量内存，内核遇到了另一个棘手的问题―初始化。

Page结构体需要初始化后才能加入到buddy分配器供内核中各个模块使用，可想而知，随着内存容量的增加，需要初始化的page结构体也会增加。如果没有记错的话，当内存达到T级别，page结构体初始化的时间将达到分钟级。

那有什么办法解决这个问题呢？有的，那就是

> 延迟满足

其实思想很简单，在系统启动时只初始化部分的page结构体，然后使用内核线程来初始化剩下的page结构体。从而达到尽量减少page结构体初始化对系统启动的影响。

多说无益，我们还是直接来看代码吧。

# 正常情况

为了有一个对比，我们先来看看没有defer_init的情况。

```c
 start_kernel()
     setup_arch()
         x86_init.paging.pagetable_init() -> paging_init()
             sparse_init()
                 sparse_init_nid()
                     map = __populate_section_memmap()        (1)
                     sparse_init_one_section(sec, map)
             zone_sizes_init()
                 free_area_init() -> free_area_init_node()
                     free_area_init_core()
                         memmap_init() -> memmap_init_zone()
                             __init_single_page()             (2)
     mm_init() -> mem_init()
         memblock_free_all()
             free_low_memory_core_early()
                 __free_memory_core() -> __free_pages_memory()
                     memblock_free_pages()
                         __free_pages_core()                  (3)
```

Page结构体要经历三个过程，被分配，被初始化，添加到buddy。

在上述代码片段中，分别标出了这三个步骤在正常情况下发生的时机。而defer_init要解决的就是2,3在初始化时占用的时间过多，导致系统启动时间过长。

既然是在2,3的地方占用了太多时间，那么就把这部分的工作延后执行把。

# 延迟满足

现在我们来看看内核代码是如何把这部分的工作延后执行的。


```c
 start_kernel()
     setup_arch()
         x86_init.paging.pagetable_init() -> paging_init()
             sparse_init()
                 sparse_init_nid()
                     map = __populate_section_memmap()
                     sparse_init_one_section(sec, map)
             zone_sizes_init()
                 free_area_init() -> free_area_init_node()
                     free_area_init_core()
                         memmap_init() -> memmap_init_zone()
                             defer_init()                     (1)
                             __init_single_page()
     mm_init() -> mem_init()
         memblock_free_all()
             free_low_memory_core_early()
                 __free_memory_core() -> __free_pages_memory()
                     memblock_free_pages()
                         early_page_uninitialised()           (2)
                         __free_pages_core()
     arch_call_rest_init() -> rest_init()
         pid = kernel_thread(kernel_init, NULL, CLONE_FS);
             kernel_init_freeable() -> page_alloc_init_late()
                 deferred_init_memmap()                       (3)
```

在上面的代码片段中可以看到，在1，2的地方分别增加了判断，是否要跳过这部分的初始化。

接着在3的位置启用了一个内核线程来初始化剩下的page结构体。

好了，这件事情就是这么简单～

[1]: /mm/52-where_is_page_struct.md
