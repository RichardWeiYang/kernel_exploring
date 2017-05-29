一般大家都比较了解内存分配的伙伴系统，但是你知道吗，当内核刚启动的时候，伙伴系统还没有准备好，是不能使用的。这个时候就有另一个简单的内存分配器--memblock。

# 看一眼历史

memblock也不是内核的原配，在memblock之前还有其他的初期内存分配器，比如bootmem。memblock是在2010年Yinghai提出的。有兴趣的可以看一下当时的[邮件列表][1]中的讨论。

而在代码中，应该是这个commit引入了memblock。

[95f72d1ed41a66f1c1c29c24d479de81a0bea36f][2]

可以看到，memblock以前叫lmb，而这个补丁只是改了下名字。

再具体的历史信息我也不是很清楚了，如果有更多好玩的信息，欢迎告诉我～

# 整体架构


memblock管理了两段区域：memblock.memory和memblock.reserved。

所有物理上可用的内存区域都会被添加到memblock.memory。而被分配或者被系统占用的区域则会添加到memblock.reserved。

**注意: 被分配的内存空间并不会从memblock.memory区域中移除。**

让我借用一张图来解释一下，这张图的原版在[这个系列][3]，也是我非常喜欢的一个内核探索的系列。

```
+---------------------------+   +---------------------------+
|         memblock          |   |       Array of the        |
|  _______________________  |   |      memblock_region      |
| |        memory         | |   |                           |
| |      memblock_type    |-|-->| [start1, end1)            |
| |_______________________| |   |                           |
|                           |   | [start2, end2)            |
|                           |   |                           |
|                           |   | [start3, end3)            |
|                           |   |                           |
|                           |   +---------------------------+
|                           |                                
|  _______________________  |   +---------------------------+
| |       reserved        | |   |       Array of the        |
| |      memblock_type    |-|-->|      memblock_region      |
| |_______________________| |   |                           |
|                           |   |  [start1, end1)           |
|                           |   |                           |
|                           |   |  [start2, end2)           |
|                           |   |                           |
+---------------------------+   +---------------------------+
```

在上图的这个例子中，当前memblock的状态是：
1. 当前系统中各有三段可用的内存空间 [start1/2/3, end1/2/3)
2. 而其中的两个已被分配 [start1/2, end1/2)

假如你想要释放 [start2, end2), 那么memblock.memory并不会有什么变化, 只要从memblock.reserved中移除 [start2, end2) 就好。

希望通过这张简单的图能帮助你理解memblock的运作原理。

PS:其实我隐藏了一个很重要的信息，不过现在暂时不需要看到他。待到春花烂漫，你们自然就会相见。

# 重要API

对memblock有了大致的了解，你或许会想知道相关的api是什么样子的，是怎么使用的。

## 添加删除内存区域

使用memblock的第一步就是要从下一层中获取可用的内存区域并填写到memblock.memory中。这是通过memblock_add() and memblock_remove()实现的。而且这两个函数保证运行后memblock中的区域是排序的。

**注意：这两个函数只改变memblock.memory。**

## 分配释放内存

memblock的重要作用就是内核初期的内存分配器了。那通过memblock来分配释放内存分别通过memblock_alloc() and memblock_free()来实现。

**注意： 这两个函数只改变memblock.reserved。**

# 获得最初的内存布局

之前我们已经看过，内存的信息通过e820从硬件中获取保存在了相应的结构体中。那现在的问题就是memblock是怎么对应上实际的物理内存的呢？

在x86平台，这个工作就交给了memblock_x86_fill()。

PS: 在最新的代码4.11版本中改成了e820__memblock_setup()。

```
void __init memblock_x86_fill(void)
{
    int i;
    u64 end;

    /*
     * EFI may have more than 128 entries
     * We are safe to enable resizing, beause memblock_x86_fill()
     * is rather later for x86
     */
    memblock_allow_resize();

    for (i = 0; i < e820.nr_map; i++) {
        struct e820entry *ei = &e820.map[i];

        end = ei->addr + ei->size;
        if (end != (resource_size_t)end)
            continue;

        if (ei->type != E820_RAM && ei->type != E820_RESERVED_KERN)
            continue;

        memblock_add(ei->addr, ei->size);
    }

    /* throw away partial pages */
    memblock_trim_memory(PAGE_SIZE);

    memblock_dump_all();
}
```

通过他就建立了硬件信息和memblock之间的联系。在x86平台上，不断从e820中获取内存底层信息，并添加到memblock中。你看是不是很简单了。

# 具体实现

## memblock_add_range()

这个函数用来添加一个区域到membloc中，比如添加一个硬件的内存区域到memblock.memroy。

实现的方法比较简单，添加再合并。总的来说一共有四种情况。

* total covered
* no overlap
* partial overlapped – beginning
* partial overlapped – end

让我们来挨个儿解释。

图例：
(base, end) 表示要添加的新的内存区域.
[rbase, rend] 表示已有的内存区域.

### total coverd

```
           (base,     end)
      [rbase,             rend]
```

在这种情况下，除了检查区域的属性，其他不会做。

### no overlap

```
    (base,     end)
                      [rbase,             rend]
```

这种情况也简单， 直接添加就好。

### partial overlapped – beginning

```
   (base,                  end)
               [rbase,             rend]

   After insert the lower part.

               ||
               ||
               vv

   （baes, end'）          (end)
               [rbase,             rend]
```

在这种情况下， (base, rbase-1) 将会被添加到区域中。此后base被设置为end。这样就回到了第一个total cover的情况。

### partial overlapped – end

```
                        (base,          end)
        [rbase,             rend]

   After truncate the lower part.

               ||
               ||
               vv

                                (base', end)
        [rbase,             rend]
```

这种情况下(base, rend)部分将会被忽略，而base会直接赋值为rend。这样也就回到了第二种情况 no overlap。

## memblock_isolate_range()

这个函数在多个地方使用到，比如memblock_remove_range() 和memblock_set_node()。这个函数的目的就是按照要求将memblock中的区域划分开。

```
                         (base,                         end)

      [rbase1,                   rend1]    [rbase2,                  rend2]

           After split

               ||
               ||
               vv

                         (base,                         end)

      [rbase1,    rend1'][rbase1',rend1]   [rbase2,   rend2'][rbas2', rend2]


                          start_rgn         end_rgn
```

该函数一共有五个参数

* type  指定是memblock.memory或memblock.reserved
* base/size 指定像划分的那段区域的范围
* start_rgn/end_rgn  返回划分完后这段区域对应的编号

其实就是划分，划分完了返回被划分的那段区域的编号～

[1]: https://lkml.org/lkml/2010/7/13/114
[2]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/mm/memblock.c?id=95f72d1ed41a66f1c1c29c24d479de81a0bea36f
[3]: https://0xax.gitbooks.io/linux-insides/content/MM/linux-mm-1.html
