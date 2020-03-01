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
