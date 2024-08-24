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


# 参考资料

* [内核文档][1]
* [作者博客][2]

[1]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/core-api/maple_tree.rst
[2]: https://blogs.oracle.com/linux/the-maple-tree

