slub确实是一个复杂的东西，究竟如何能把这个东西讲明白也是很头大的事情。今天我尝试用图示来解释一下slub的运作过程。或许会对你的理解有点帮助。

既然是内存管理难免离不开**创建**、**分配**、**回收**。那我就从这几个方面展示一下slub的动态情况。

# 创建

所谓创建就是创建kmem_cache这个结构体，可以认为这个结构体就是slub分配时的一个中心节点。你可以在这个节点上获得所有这个slub的信息。

比如在上一篇中看到的object_size,size,oo等。这些数据在创建后就不变了，主要用来存储本slub的大小信息，所以就不在这篇文章中列出。

在这篇文章中我们主要关注的是

    * cpu_slab
    * node

可以看到，在slub中又分了两级来管理内存：per cpu层级和NUMA node层级。可以认为这是分层思想的又一次运用。

下图中显示了kmem_cache刚创建时的样子，可以看到什么都是空的，一切都刚刚开始。

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

# 分配

创建完成后就是要分配了。由于采用了分层的设计，分配时根据每个层次是否存在内存分成了几种情况。

在系统运行过程中，按照一下顺序查看是否拥有可用内存。

    * cpu_slab->freelist
    * cpu_slab->partial
    * cpu_slab->node

原理也很简单，就是先看最方便拿到的空间中是否有内存。

接下来我就按照不同的情况，给大家看看slub运作时的动态。

## 哪都没内存

按照以上的顺序，这个过程应该是以上三个空间中都没有内存的情况。不过这也是slub刚创建后，第一次分配时的情况。所以我干脆就把这个当作第一种情形了。

做的工作也很简单，就是从buddy系统中拿到一个page，装到cpu_slab->freelist中。这一步在allocate_slab()中执行。

这种情形的初始状态是大家都为空，但是下图中我仍然画了两个状态。**目的是为了突出page->freelist指针的变化。**

```
    kmem_cache                      
    +------------------------------+
    |name   (char *)               |
    +------------------------------+
    |cpu_slab                      |  * per cpu variable pointer
    |   (struct kmem_cache_cpu*)   |
    |   +--------------------------+
    |   |stat[NR_SLUB_STAT_ITEMS]  |
    |   |  (unsigned)              |
    |   +--------------------------+
    |   |tid      (unsigned long)  |
    |   |                          |
    |   |freelist                  |                       
    |   |    (void **)             |                       
    |   |                          |                       
    |   |page                   +--|--->+--------+--------+--------+--------+
    |   |   (struct page *)     |  |    |        |        |        |        |
    |   |   +-------------------|--+    +--------+--------+--------+--------+
    |   |   |freelist        ---+  |
    |   |   |inuse = objects       |
    |   |   |frozen = 1            |
    |   |   +----------------------+
    |   |partial                   |  = NULL
    |   |   (struct page *)        |
    +---+--------------------------+

    kmem_cache                      
    +------------------------------+
    |name   (char *)               |
    +------------------------------+
    |cpu_slab                      |  * per cpu variable pointer
    |   (struct kmem_cache_cpu*)   |
    |   +--------------------------+
    |   |stat[NR_SLUB_STAT_ITEMS]  |
    |   |  (unsigned)              |
    |   +--------------------------+
    |   |tid      (unsigned long)  |
    |   |                          |
    |   |freelist            ------|----+  = page->freelist
    |   |    (void **)             |    |
    |   |                          |    v
    |   |page                      |    +--------+--------+--------+--------+
    |   |   (struct page *)        |    |        |        |        |        |
    |   |   +----------------------+    +--------+--------+--------+--------+
    |   |   |freelist = NULL       |
    |   |   |inuse = objects       |
    |   |   |frozen = 1            |
    |   |   +----------------------+
    |   |partial                   |  = NULL
    |   |   (struct page *)        |
    +---+--------------------------+
```

## freelist上有内存

这种情形也是被称之为Fast path的情形，因为这中情况下，只要移动freelist指针即可。这步在slab_alloc_node()中执行。

在下图中，freelist就从a移动到了A。

**需要注意的是freelist中保存的是一个链表而不是线性表，所以指针的变化不一定是后移。**

```
    kmem_cache                      
    +------------------------------+
    |name   (char *)               |
    +------------------------------+
    |cpu_slab                      |  * per cpu variable pointer
    |   (struct kmem_cache_cpu*)   |
    |   +--------------------------+
    |   |stat[NR_SLUB_STAT_ITEMS]  |
    |   |  (unsigned)              |
    |   +--------------------------+
    |   |tid      (unsigned long)  |
    |   |                          |             a                 A
    |   |freelist            ------|-------------+..................     
    |   |    (void **)             |             |                 .
    |   |                          |             v                 v
    |   |page                      |    +--------+--------+--------+--------+
    |   |   (struct page *)        |    |        |        |        |        |
    |   |   +----------------------+    +--------+--------+--------+--------+
    |   |   |freelist = NULL       |
    |   |   |inuse = objects       |
    |   |   |frozen = 1            |
    |   |   +----------------------+
    |   |                          |
    |   |partial                   |  = NULL
    |   |   (struct page *)        |
    +---+--------------------------+
```

## partial有内存

以此类推，接下来的情形就是partial上有内存。这时，我们就需要从partial上找到可用的page，装到freelist上。这步在___slab_alloc() -> get_freelist()执行。

下面两个图分别显示了装之前和之后的变化，可以看到原来在partial中的page A，安装后到了cpu_slub->page。

**值得注意的是在partial上的page，frozen也是1，但是inuse则反映真实的分配情况而不是等于objects了。**

```
    kmem_cache                      
    +------------------------------+
    |name   (char *)               |
    +------------------------------+
    |cpu_slab                      |  * per cpu variable pointer
    |   (struct kmem_cache_cpu*)   |
    |   +--------------------------+
    |   |stat[NR_SLUB_STAT_ITEMS]  |
    |   |  (unsigned)              |
    |   +--------------------------+
    |   |tid      (unsigned long)  |
    |   |                          |
    |   |freelist                  |  = NULL
    |   |    (void **)             |
    |   |page                      |  = NULL
    |   |   (struct page *)        |
    |   |                          |
    |   |partial             ------|--->+-----------+     +-----------+
    |   |   (struct page *)        |    |A      next|---->|B      next|
    |   |                          |    +-----------+     +-----------+
    |   |                          |    |pobjectsA  |  =  |pobjectsB  | + objectsA
    |   |                          |    |           |     |           |
    |   |                          |    |inuse <    |     |inuse <    |
    |   |                          |    | objectsA  |     | objectsB  |
    |   |                          |    |frozen = 1 |     |frozen = 1 |
    |   |                          |    +-----------+     +-----------+
    +---+--------------------------+

    kmem_cache                      
    +------------------------------+
    |name   (char *)               |
    +------------------------------+
    |cpu_slab                      |  * per cpu variable pointer
    |   (struct kmem_cache_cpu*)   |
    |   +--------------------------+
    |   |stat[NR_SLUB_STAT_ITEMS]  |
    |   |  (unsigned)              |
    |   +--------------------------+
    |   |tid      (unsigned long)  |
    |   |                          |
    |   |freelist            ------|----+   = page->freelist
    |   |    (void **)             |    |
    |   |                          |    v
    |   |page                      |    +--------+--------+--------+--------+
    |   |   (struct page *)  A     |    |        |        |        |        |
    |   |   +----------------------+    +--------+--------+--------+--------+
    |   |   |freelist = NULL       |
    |   |   |inuse = objects       |
    |   |   |frozen = 1            |
    |   |   +----------------------+
    |   |                          |
    |   |partial             ------|--->+-----------+
    |   |   (struct page *)        |    |B      next|
    |   |                          |    +-----------+
    +---+--------------------------+
```

## node上有内存

看完了freelist/partial，最后找到在cpu_slub->node上有内存。那只能从node上把page拿下来，装到freelist上。这个过程在new_slab_objects()->get_partial()->get_partial_node()中执行。

在这里需要注意，在某些情况下，不仅会装到freelist上，还会多拿几个page装到partial上。节省了下次又找不到的情况。在下图中显示的page B就从node链表装到了partial上。而且在node上的page中frozen为0。

```
    kmem_cache                      
    +------------------------------+
    |name   (char *)               |
    +------------------------------+
    |cpu_slab                      |  * per cpu variable pointer
    |   (struct kmem_cache_cpu*)   |
    |   +--------------------------+
    |   |stat[NR_SLUB_STAT_ITEMS]  |
    |   |  (unsigned)              |
    |   +--------------------------+
    |   |tid      (unsigned long)  |
    |   |                          |
    |   |freelist                  |  = NULL
    |   |    (void **)             |
    |   |page                      |  = NULL
    |   |   (struct page *)        |
    |   |                          |
    |   |partial                   |  = NULL
    |   |   (struct page *)        |
    |   |                          |
    +---+--------------------------+
    |node[MAX_NUMNODES]            |
    |   (struct kmem_cache_node*)  |
    |   +--------------------------+
    |   |nr_partial  = 3           |
    |   |    (unsigned long)       |
    |   |partial               ----|--->+-----------+     +-----------+     +-----------+
    |   |    (struct list_head)    |    |A       lru|---->|B       lru|---->|        lru|
    +---+--------------------------+    +-----------+     +-----------+     +-----------+
                                        |inuse <    |     |inuse <    |
                                        | objects   |     | objects   |
                                        |frozen = 0 |     |frozen = 0 |
                                        +-----------+     +-----------+

    kmem_cache                      
    +------------------------------+
    |name   (char *)               |
    +------------------------------+
    |cpu_slab                      |  * per cpu variable pointer
    |   (struct kmem_cache_cpu*)   |
    |   +--------------------------+
    |   |stat[NR_SLUB_STAT_ITEMS]  |
    |   |  (unsigned)              |
    |   +--------------------------+
    |   |tid      (unsigned long)  |
    |   |                          |
    |   |freelist            ------|----+   = page->freelist
    |   |    (void **)             |    |
    |   |                          |    v
    |   |page                      |    +--------+--------+--------+--------+
    |   |   (struct page *)  A     |    |        |        |        |        |
    |   |   +----------------------+    +--------+--------+--------+--------+
    |   |   |freelist = NULL       |
    |   |   |inuse = objects       |
    |   |   |frozen = 1            |
    |   |   +----------------------+
    |   |                          |
    |   |partial             ------|--->+-----------+
    |   |   (struct page *)        |    |B      next|
    |   |                          |    +-----------+
    |   |                          |    |pobjects   |  = objects
    |   |                          |    |           |
    |   |                          |    |inuse <    |
    |   |                          |    | objects   |
    |   |                          |    |frozen = 1 |
    |   |                          |    +-----------+
    |   |                          |
    +---+--------------------------+
    |node[MAX_NUMNODES]            |
    |   (struct kmem_cache_node*)  |
    |   +--------------------------+
    |   |nr_partial  = 1           |
    |   |    (unsigned long)       |
    |   |partial               ----|--->+-----------+
    |   |    (struct list_head)    |    |        lru|
    +---+--------------------------+    +-----------+
```
