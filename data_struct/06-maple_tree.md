Maple Tree是2022年，由Liam引入，为了解决vma处理时的锁问题。

```
commit 54a611b605901c7d5d05b6b8f5d04a6ceb0962aa
Author: Liam R. Howlett <Liam.Howlett@Oracle.com>
Date:   Tue Sep 6 19:48:39 2022 +0000

    Maple Tree: add new data structure
    
    Patch series "Introducing the Maple Tree"
```

虽然Maple Tree脱胎于B树，但是细节上还是有很大的差异的。而且这代码我真的是费了老大劲才终于看懂一点。赶紧记录一下，以免忘记。

# maple tree的目标

引入新的数据结构总是有目的的，在上面的这个commit中，Liam叙述了这个目标：

  * 减少访问时的cache miss
  * 降低mmap_lock的竞争

减少cache miss的原因是：

  * maple tree的分支比红黑树多，所以高度低。这样要找到某个地址空间需要检索的节点数就少。
  * 访问相邻vma的链表是嵌入在vma结构体内的，所以每次访问都要再次读取链表。但用了maple tree后，这个信息已经保存在访问过的节点内了。

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

## mas_wr_walk()

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

## mas_wr_node_store()

遍历完找到需要修改的叶子节点，我们最终的目的是将[index, last]写入到节点中。这个过程有好几种case，我们打开看一下。

首先按照offset和offset_end的关系，分两个大类。也就是新的区域是在原来的一个区域里，还是跨了多个区域。
然后按照[index, last]和[r_min, end_piv]的关系，分四小类。决定是改写还是要插入新的。

因为有新增插入slot的情况，此时新增一个变量new_end以表示完成插入后节点预期长度。

### offset == offset_end

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

#### index == r_min && last == r_max

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

#### index != r_min && last != r_max

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

#### index == r_min && last != r_max

```
    +------+......+------+......+------+......+------+
    |  S0  |  P0  | entry| last |  S1  |  P1  |  S2  |
    +------+......+------+......+------+......+------+
    |             |             |                    |
    |<- copied  ->|<-modified ->|<---    copied  --->|

 new_end = end + 1
```

#### index != r_min && last == r_max

```
    +------+......+------+......+------+......+------+
    |  S0  |  P0  |  S1  | idx-1| entry|  P1  |  S2  |
    +------+......+------+......+------+......+------+
    |             |                           |      |
    |<- copied  ->|<---     modified      --->|< cp >|

 new_end = end + 1
```

### offset < offset_end

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

#### index == r_min && last == end_piv

```
    +------+......+------+......+------+
    |  S0  |  P0  | entry|  Pe  |  Sx  |
    +------+......+------+......+------+
    |             |             |      |
    |<- copied  ->|<- modified >|< cp >|

 new_end = end - (offset_end - offset)
```

#### index != r_min && last != end_piv

```
    +------+......+------+......+------+......+------+......+------+
    |  S0  |  P0  |  S1  | idx-1| entry| last |  Se  |  Pe  |  Sx  |
    +------+......+------+......+------+......+------+......+------+
    |             |                           |                    |
    |<- copied  ->|<---    modified       --->|<---    copied  --->|

 new_end = end + 2 - (offset_end - offset)
```

#### index == r_min && last != end_piv

```
    +------+......+------+......+------+......+------+
    |  S0  |  P0  | entry| last |  Se  |  Pe  |  Sx  |
    +------+......+------+......+------+......+------+
    |             |             |                    |
    |<- copied  ->|<-modified ->|<---    copied  --->|

 new_end = end + 1 - (offset_end - offset)
```

#### index != r_min && last == end_piv

```
    +------+......+------+......+------+......+------+
    |  S0  |  P0  |  S1  | idx-1| entry| last |  Sx  |
    +------+......+------+......+------+......+------+
    |             |                           |      |
    |<- copied  ->|<---     modified      --->|< cp >|

 new_end = end + 1 - (offset_end - offset)
```

## mas_wr_modify()

mas_wr_modify()是mas_wr_node_store()的父函数，虽然在mas_wr_node_store()中可以处理（几乎）所有的情况，但是在mas_wr_modify()里还是做了点优化。

原因是mas_wr_node_store()的操作都是基于复制节点来实现的。而在某些情况下，并不需要复制节点，而只要在原节点上进行修改依然可以做到rcu safe。

在了解了各种修改的情况后，我们再来看看这些可以优化的case。

**PS: 强调一点，进入mas_wr_modify时，mas->node一定是叶子节点。所以不用拷贝gap。**

### offset == end: mas_wr_append()

变化的区间正好是当前节点最后一个区域的情况。而且隐含的条件是offset == offset_end，因为offset_end不可能大于end。

因为只需要添加到节点最后，所以单独拿出来做了一个优化。不过对rcu情况不适用。

### new_end == end: mas_wr_slot_store()

这中情况说明节点内的数据长度并没有变化，所以我们不需要复制一个新的节点来做变更，只要在本地做修改就性了。

而且这里有个隐含条件是offset < offset_end，因为在offset == offset_end的case中，没有长度不变的情况。PS:唯一不便的情况已经被特殊处理了。

# 常用API

maple tree提供了一些API，然后其他子系统，比如mm，会在这个基础上再封装一层。

## mas_prev|next()

这一类一共有四个函数

* mas_prev(mas, min)
* mas_next(mas, max)
* mas_prev_range(mas, min)
* mas_next_range(mas, max)

他们两两行为是对称的，一个是往前，一个是往后。

> 在叶子节点一层，在mas->index位置上往前|后一格。

区别在与

* mas_prev|next(): 是要一直找到前|后面一格非空的slot。
* mas_prev|next_range(): 则只是往前|后一格，如果是空也没有关系。


除此之外，还要关注mas中index/last的变化。

* 遍历时是按照mas->index的值去找的，和mas->last没有关系。查找的最大范围是通过参数max来指定的。
* 不论找不找得到，mas->index/last都会设置到某个真实的range的范围上
* 如果不能再往前或者往后了，status会被设置成ma_overflow或者ma_underflow
* 最后mas->index/last指定的范围一定和min/max指定的范围有交叉

## mas_find()

这个函数的行为和mas_next()非常相似，除了在第一次执行的时候。

如果第一次执行时，mas->index对应有值则返回这个值。而mas_next()则需要时返回下一个。
如果第一次执行时，mas->index对应值为NULL，返回结果和mas_next()一样。

# 测试代码

在内核代码里有专门的测试程序，在这里记录一下。一旦有改动，需要通过测试程序才能提交。

```
# cd tools/testing/radix-tree/
# make maple
# ./maple
```

还有对应的内核模块测试代码。

```
# ls lib/test_maple_tree.c
```

不过需要打开选项CONFIG_TEST_MAPLE_TREE。

编译完，安装内核模块也是测试。不过比用户态的程序少了点case。

# 参考资料

* [内核文档][1]
* [作者博客][2]

[1]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/core-api/maple_tree.rst
[2]: https://blogs.oracle.com/linux/the-maple-tree

