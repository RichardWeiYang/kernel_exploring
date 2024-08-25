Maple Tree是2022年，由Liam引入，为了解决vma处理时的锁问题。

```
commit 54a611b605901c7d5d05b6b8f5d04a6ceb0962aa
Author: Liam R. Howlett <Liam.Howlett@Oracle.com>
Date:   Tue Sep 6 19:48:39 2022 +0000

    Maple Tree: add new data structure
    
    Patch series "Introducing the Maple Tree"
```

虽然Maple Tree脱胎于B树，但是细节上还是有很大的差异的。而且这代码我真的是费了老大劲才终于看懂一点。赶紧记录一下，以免忘记。

# maple_node如何表达数据

Maple Tree中的一个节点用maple_node表示，理解了其中时如何表达数据的，才能理解相关的操作是如何执行的。

```
 S: slot
 P: pivot (P0 < P1 < P2 ...)

 +------+......+------+......+------+
 |  S0  |  P0  |  S1  |  P1  |  S2  |
 +------+......+------+......+------+

 [min, P0]     [P0+1, P1]    [P1+1, max]
```

每个节点包含了多个slot/pivot，其中pivot按照递增的顺序排列。
当我们在一个坐标上依次标注pivot后，就展现出一段段由pivot分割的区域，而这些区域对应的值，则存储在相应的slot中。

如果我们以slot为中心，每一个slot[n]代表的区域就是[P(n-1) + 1, Pn]。当然其中由两个特例，头和尾的slot由整个节点的min/max限制。

便于理解，我们看一下父子关系下的情况。

```
         +------+......+------+......+------+
         |  S0  |  P0  |  S1  |  P1  |  S2  |
         +------+......+------+......+------+
                          ^
                         / parent
                       /
   +------+......+------+......+------+
   |  Sa  |  Pa  |  Sb  |  Pb  |  Sc  |
   +------+......+------+......+------+
   [P0+1, Pa]    [Pa+1, Pb]    [Pb+1, P1]

```

区域slot 1代表的区域，分割成了更小粒度的区域，由子节点表达。此时子节点的min/max就是slot 1的区域界限P0+1和P1了。


# 函数详解

# mas_wr_walk()

这个函数是用来在写入数据前查找对应节点的，这个的作用和红黑树和B树在插入前做的动作一样。所以遍历过程就不再赘述，而是讲一下遍历中保存的几个在最后修改值过程中要用到的值。

我们先来直观看一下遍历完，几个重要值之间的关系

```
 mas->node
 +------+......+------+......+------+......+------+......+------+
 |  S0  |  P0  |  S1  |  P1  |  S2  |  P2  |  S3  |  P3  |  S4  |
 +------+......+------+......+------+......+------+......+------+
 [mas->min,                                             mas->max]
               [r_min, r_max]           [, end_piv]
               [index,                        last]

                   ^                           ^
                   |                           |
                offset                    offset_end
```

依次解释一下：

  * 遍历过程中mas->node指向的是当前的节点，遍历完时就指向需要修改的叶子节点
  * mas->min/max表达的是当前节点的界限
  * mas->index/last形成了一个区域[index, last]，这是本次修改涉及的区域
  * offset/offset_end是[index, last]这个区域所在的起始和结束的下标
  * [r_min, r_max]是offset下标对应区域范围
  * end_piv是offset_end下标对应区域右侧界限

根据上面的这些含义，我们可以得出的隐含含义是：

  * 区域[index, last] 包含在 区域[r_min, end_piv]

有了这些背景知识，我们就可以看看具体的修改过程，以及各种不同的case了。

PS: 补充一点， end_piv是在mas_wr_end_piv()中计算的，和mas_wr_walk()搭配使用才有效果。

# mas_wr_node_store()

遍历完找到需要修改的叶子节点，我们最终的目的是将[index, last]写入到节点中。这个过程有好几种case，我们打开看一下。

首先按照offset和offset_end的关系，分两个大类。也就是新的区域是在原来的一个区域里，还是跨了多个区域。
然后按照[index, last]和[r_min, end_piv]的关系，分四小类。决定是改写还是要插入新的。

因为有新增插入slot的情况，此时新增一个变量new_end以表示完成插入后节点预期长度。

## offset == offset_end

这时，说明[index, last]这个区域就在offset下标所指定的区域内，并不影响到其他区域。
并且, end_piv实际上等于r_max。

此时我们假设初始情况如下：

```
    +------+......+------+......+------+
    |  S0  |  P0  |  S1  |  P1  |  S2  |
    +------+......+------+......+------+

                      ^
                      |
                   offset
```

### index == r_min && last == r_max

这种情况说明只要直接改写原有slot的值，并没有改变区域划分。比较直接。

```
    +------+......+------+......+------+
    |  S0  |  P0  | entry|  P1  |  S2  |
    +------+......+------+......+------+
                  |      |
                  |      |
		  direct replace

 new_end = end
```

此时节点长度并没有变化。

### index != r_min && last != r_max

这种情况需要在原区域的前后都要切一块出来。

```
    +------+......+------+......+------+......+------+......+------+
    |  S0  |  P0  |  S1  | idx-1| entry| last |  S1  |  P1  |  S2  |
    +------+......+------+......+------+......+------+......+------+
    |             |                           |                    |
    |<- copied  ->|<---    modified       --->|<---    copied  --->|

 new_end = end + 2
```

所以这种情况下，需要增加两个slot来标记。

### index == r_min && last != r_max

```
    +------+......+------+......+------+......+------+
    |  S0  |  P0  | entry| last |  S1  |  P1  |  S2  |
    +------+......+------+......+------+......+------+
    |             |             |                    |
    |<- copied  ->|<-modified ->|<---    copied  --->|

 new_end = end + 1
```

### index != r_min && last == r_max

```
    +------+......+------+......+------+......+------+
    |  S0  |  P0  |  S1  | idx-1| entry|  P1  |  S2  |
    +------+......+------+......+------+......+------+
    |             |                           |      |
    |<- copied  ->|<---     modified      --->|< cp >|

 new_end = end + 1
```

## offset < offset_end

这时，说明[index, last]在原节点上跨了多个区域。并且在小case的比较上，不是对比r_max了，而是end_piv。


此时我们假设初始情况如下：
```
    +------+......+------+......+       +------+......+------+
    |  S0  |  P0  |  S1  |  P1  |  xxx  |  Se  |  Pe  |  Sx  |
    +------+......+------+......+       +------+......+------+

                      ^                    ^
                      |                    |
                   offset               offset_end
```

### index == r_min && last == end_piv

```
    +------+......+------+......+------+
    |  S0  |  P0  | entry|  Pe  |  Sx  |
    +------+......+------+......+------+
    |             |             |      |
    |<- copied  ->|<- modified >|< cp >|

 new_end = end - (offset_end - offset)
```

### index != r_min && last != end_piv

```
    +------+......+------+......+------+......+------+......+------+
    |  S0  |  P0  |  S1  | idx-1| entry| last |  Se  |  Pe  |  Sx  |
    +------+......+------+......+------+......+------+......+------+
    |             |                           |                    |
    |<- copied  ->|<---    modified       --->|<---    copied  --->|

 new_end = end + 2 - (offset_end - offset)
```

### index == r_min && last != end_piv

```
    +------+......+------+......+------+......+------+
    |  S0  |  P0  | entry| last |  Se  |  Pe  |  Sx  |
    +------+......+------+......+------+......+------+
    |             |             |                    |
    |<- copied  ->|<-modified ->|<---    copied  --->|

 new_end = end + 1 - (offset_end - offset)
```

### index != r_min && last == end_piv

```
    +------+......+------+......+------+......+------+
    |  S0  |  P0  |  S1  | idx-1| entry| last |  Sx  |
    +------+......+------+......+------+......+------+
    |             |                           |      |
    |<- copied  ->|<---     modified      --->|< cp >|

 new_end = end + 1 - (offset_end - offset)
```

# 参考资料

* [内核文档][1]
* [作者博客][2]

[1]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/core-api/maple_tree.rst
[2]: https://blogs.oracle.com/linux/the-maple-tree

