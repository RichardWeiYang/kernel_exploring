先抄一段来自度娘的定义：

> Swap分区在系统的运行内存不够用的时候，把物理内存中的一部分空间释放出来，以供当前运行的程序使用。

也就是系统提供了一种方法，当系统内存吃紧（或者在必要的时候），可以将内存内容保存到指定的文件上。这样释放出来的内存空间，就可以给到系统其他进程使用了。

本章我们就来看看这个swap工作的基本概念。

# 开启一个swap分区

swap并不是系统中必须的，也就是我们可以不用swap。那么此时当系统内存吃紧时，我估计会把内存扔掉吧。

那如何开启呢？ 下面给出一个简单的例子：

```
# create a file with 256M
sudo dd if=/dev/zero of=myswap bs=32k count=8192
# format myswap to swap file
sudo mkswap myswap
# enable it
sudo swapon myswap
```

怎么样，是不是很简单。我们只要指定一个文件，格式化成swap分区。这样就可以通过swapon系统调用将一个swap分区用起来了。

可以通过很多命令来获取swap分区的使用情况，如top, htop, cat /proc/swaps。不过信息比较简单。

# 初始化一个swap分区

要使用一个swap分区，那就要对这个分区初始化。这个初始化的过程有点长，但是从数据结构的角度来看也不算太复杂。让俺尝试用下面的图来表示。

```
                                         0     1              nr_swapfiles       MAX_SWAPFILES -1
                                         |     |                 |                 |
                                         v     v                 v                 v
        struct swap_info_struct       +-----+-----+-----+-----+-----+-----+-----+-----+
         *swap_info[MAX_SWAPFILES]    |     |     |     |     |     |     |     |     |
                                      |     |     |     |     |     |     |     |     |
                                      +-----+-----+-----+-----+-----+-----+-----+-----+
        struct address_space          +-----+-----+-----+-----+-----+-----+-----+-----+
      *swapper_space[MAX_SWAPFILES]   |     |     |     |     |     |     |     |     |
                                      |     |     |     |     |     |     |     |     |
                                      +-----+-----+-----+-----+-----+-----+-----+-----+
        unsigned int                  +-----+-----+-----+-----+-----+-----+-----+-----+
      nr_swapper_spaces[MAX_SWAPFILES]|     |     |     |     |     |     |     |     |
                                      |     |     |     |     |     |     |     |     |
                                      +-----+-----+-----+-----+-----+-----+-----+-----+
                                                     ^
                                                     |
                                                     |
                                                     +----------------------+
                                                                            |
                                                                            |
                                                                            |
                                      struct swap_info_struct               |
                                      +--------------------------------+    |
                                      |type                         ---|----+
                                      |                                | index in swap_info,
                                      |                                | swapper_spaces, nr_swapper_spaces
                                      |                                |
                                      |flags                           | SWP_USED
                                      |                                |
                                      |prio                            |
                                      |    (signed short)              |
     swap_active_head      <--->      |list                            |
     swap_avail_heads[]    <--->      |avail_lists[0]                  |
                                      |    (struct plist_node)         |
                                      +--------------------------------+
```

对这张图有几点可以说：

* 每个swap分区由一个swap_info_struct结构体表示
* 这个结构体的指针将保存在swap_info[]数组中，下标是swap_info_struct->type。（神奇的名字）
* 对应的还有两个数组swapper_space[]和nr_swapper_spaces[]，用来保存对应的address_space指针。可以一个swap分区可以对应多个address_space。
* swap_info_struct又会链接在两个优先级队列swap_active_head和swap_avail_heads[]上。看注释，前者用于swapoff，后者用于找到合适的磁盘空间。

有了大致的数据结构概念，我们可以来看看如何把内存保存到swap分区的。

# 找到空闲的swap空间

要把内存保存到某个地方，要分成两步：

* 找到可以存放的地方
* 放进去

要了解怎么找到一个空闲的空间，就要再看一眼swap_info_struct这个数据结构。

```

           struct swap_info_struct
           +--------------------------------+
           |max                             | = maxpages
           |pages                           | = nr_good_pages real available pages
           |inuse_pages                     |
           |                                |
           |highest_bit                     |    ----------------------+
           |lowest_bit                      |    ----+                 |
           |    (unsigned int)              |        |                 |
           |                                |  0     v                 v  maxpages
           |swap_map                        | [  |  |  |  |  |  |  |  |  |  ]
           |    (char *)                    |               ^
           |                                |               |
           |cluster_next                    |    -----------+     indicate next allocation
           |    (unsigned int)              |
           |                                |
           +--------------------------------+

```

出了前面那张图中展示的信息，swap_info_struct结构体中还包含了

* max:         整个swap分区页面个数
* pages:       整个swap分区可用的页面个数
* inuse_pages: 已经用掉了多少页面
* swap_map:    对每一个页面都有一个字节表示状态

接下来三个都是围绕着swap_map数组的下标：

* lowest_bit/highest_bit: 当前可用范围的最大最小下标
* cluster_next:           建议的下次可使用的下标

按照我现在的理解，这个过程就是当有请求时就从cluster_next下标去找swap_map中第一个空闲的块。然后把这个块分配出去。

具体的算法在scan_swap_map_slots()中，还有写不是特别清楚。待后续了解后再来补充。

# 写入swap分区

找到了swap分区中对应的块，接下来就要把我们想要释放的内存写入。这样才能最后释放对应的内存。

个人感觉这个动作可以在多个地方完成，先找出其中之一记录一下：

```
  shrink_page_list()
    ...
    add_to_swap()
      entry = get_swap_page()                        (1)
        ...
        get_swap_pages()
      add_to_swap_cache(page, entry, GFP_xxx)        (2)
    mapping = page_mapping(page)
    pageout(page, mapping)
      mapping->a_ops->writepage(page, wbc)           (3)
```

此处省去众多，只留下我认为比较关键的步骤。

* (1) 从swap分区中找到一个空闲块，这里还有些优化暂时跳过
* (2) 将页加入到address_space中，就是swapper_space[]数组中保存的
* (3) 调用address_space对应的函数将页写入

这里因为是一个swapfile，所以对应的a_ops就是swap_aops。在函数init_swap_address_space()中初始化。

到这里基本算是了解了swap分区的基本工作流程，当然还有更多细节有待挖掘。

# 基本概念

到这里我们要补充几个基本概念，了解一下重要的几个常用数据结构的样子。

## swap_count

在“找到空闲的swap空间”这部分，我们可以看到swap_map这个数组用来保存对应页面的状态信息。包括了：

* 被映射的数目
* 是否有cache
* 是不是坏的

这这一小节我们主要来看swap_count是如何存储**被映射的数目**的。

在 commit 570a335b8e22 swap_info: swap count continuations中，引入了采用链式的存储数目的方法。这样呢就导致了swap_map这个字节有了两种格式：

```
    First swap_map:

        COUNT_CONTINUED
         |    SWAP_HAS_CACHE
         |     |
         v     v
       +----+----+----+----+----+----+----+----+
       |    |    |    |    |    |    |    |    |
       +----+----+----+----+----+----+----+----+

    Continuations:

        COUNT_CONTINUED
         |
         |
         v
       +----+----+----+----+----+----+----+----+
       |    |    |    |    |    |    |    |    |
       +----+----+----+----+----+----+----+----+
```

总的来说，也就是第一个swap_map的格式和后续的会不一样。差别主要是第一个字节中的SWAP_HAS_CACHE被占用来表达了一个特殊含义，而后续的swap_map可以有7个bit来表达数目信息。
而每个swap_map的最高位COUNT_CONTINUED用来表示后续是否还有更高位的信息。

这样一来 后续swap_map能表达的数目信息由低7位表达，也就是     SWAP_CONT_MAX   0x7f(0111 1111)。
但是呢，第一个swap_map能表达的数目信息却不是低6位的全部，而是 SWAP_MAP_MAX    0x3e(0011 1110)。因为SWAP_MAP_BAD    0x3f(0011 1111)用来表达一个特殊含义。

所以这些定义确实有点让人需要花一些时间去理解。我把这些特殊的定义稍微总结一下：

```
Bit indication in first swap_map:

   SWAP_HAS_CACHE  0x40(0100 0000)

Bit indication in all swap_map(?):

   COUNT_CONTINUED 0x80(1000 0000)

Special value in first swap_map:

   SWAP_MAP_MAX    0x3e(0011 1110)
   SWAP_MAP_BAD    0x3f(0011 1111)
   SWAP_MAP_SHMEM  0xbf(1011 1111)

Special value in swap_map continuation:

   SWAP_CONT_MAX   0x7f(0111 1111)
```

其实呢，了解了数据结构，就基本能理解后续代码是如何运作的了。这部分主要的函数是swap_count_continued()，这个函数相当于一个对于swap_map的加减法。而函数add_swap_count_continuation()则是在需要的时候扩展相对应的计数空间。当然里面用的招数是链接page->lru，有点高难度了。

# swp_entry

这个swp_entry就是一个索引，用它就可以找到对应的swap_map和swap cache中保存的那个page。

总的来说，这个swp_entry就是将swap_info_struct->type和offset编码存放起来。大致格式如下图所示

```
    +-+-------------+---------------------------------+
    | |             |                                 |
    +-+-------------+---------------------------------+
      ^             ^                                 ^
      | type(5bits) |       index/offset              |
```

具体将type和offset编码，以及从swp_entry解码的函数为：

编码: (type, offset) -> swp_entry

>  entry = swp_entry(type, offset)

解码: swp_entry -> (type, offset)

>  type = swp_type(entry)
>  offset = swp_offset(entry)

而这个swp_entry在把page释放到设备的时候就会写到pte中。所以这是一个非常重要的结构。

此外对其中的type部分要进一步展开，这个type占用5个bit，但是真正能用来表示swap设备的没有这么多。这个值如下定义：

```
#define MAX_SWAPFILES \
	((1 << MAX_SWAPFILES_SHIFT) - SWP_DEVICE_NUM - 	SWP_MIGRATION_NUM - SWP_HWPOISON_NUM)
```

也就是这些位中会拿出一部分SWP_DEVICE_NUM, SWP_MIGRATION_NUM, SWP_HWPOISON_NUM来表示某些对页做的特殊的操作。比如对于迁移的页面，就会用SWP_MIGRATION_READ|WRITE来表示。

# 演进

基本概念差不多是上面描述的样子了，接下来我们看看更加具体的细节。而这些细节大多是一步步演进过来的。
