
```
            kmem_cache                      
            +------------------------------+
            |name                          |
            |    (char *)                  |
            +------------------------------+
            |cpu_slab                      |  * per cpu variable pointer
            |   (struct kmem_cache_cpu*)   |
            |   +--------------------------+
            |   |stat[NR_SLUB_STAT_ITEMS]  |
            |   |  (unsigned)              |
            |   |tid                       |    #cpu
            |   |          (unsigned long) |
            |   |freelist  (void **)       |    NULL
            |   |page      (struct page *) |    NULL
            |   |partial   (struct page *) |    NULL
            +---+--------------------------+
            |node[MAX_NUMNODES]            |
            |   (struct kmem_cache_node*)  |
            |   +--------------------------+
            |   |list_lock                 |
            |   |    (spinlock_t)          |
            |   |nr_partial                |    0
            |   |    (unsigned long)       |
            |   |partial                   |    empty
            |   |    (struct list_head)    |
            +---+--------------------------+
```
