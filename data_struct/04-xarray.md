现在我们来看看xarray这个数据结构。

# 眼见为实

xarray这个数据结构主要由两个结构体

  * xarray
  * xa_node

其本身并不复杂，但是组合起来究竟是什么样子呢？ 那就让我们来看看当XA_CHUNK_SHIFT = 4时，一个xarray会是什么样子。

```
xarray->xa_head = xa_node0
                  +-------------------------------+
                  |parent   = NULL                |
                  |shift    = 8                   |
                  |max_index= (1 << (8 + 4)) - 1  |
                  |offset                         |
                  |                               |
                  |slots[XA_CHUNK_SIZE]           |
                  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                  |f|e|d|c|b|a|9|8|7|6|5|4|3|2|1|0|
                  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                       |                         |
                       |                         |
                       v                         v
       xa_node2                                  xa_node1
       +-------------------------------+         +-------------------------------+
       |parent   = xa_node0            |         |parent   = xa_node0            |
       |shift    = 4                   |         |shift    = 4                   |
       |max_index= (1 << (4 + 4)) - 1  |         |max_index= (1 << (4 + 4)) - 1  |
       |offset   = d                   |         |offset   = 0                   |
       |                               |         |                               |
       |slots[XA_CHUNK_SIZE]           |         |slots[XA_CHUNK_SIZE]           |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+         +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |f|e|d|c|b|a|9|8|7|6|5|4|3|2|1|0|         |f|e|d|c|b|a|9|8|7|6|5|4|3|2|1|0|
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+         +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                                    |                         |
                                    |                         +--> [0x090, 0x09f]
                                    v
                    xa_node3
                    +-------------------------------+
                    |parent   = xa_node2            |
                    |shift    = 0                   |
                    |max_index= (1 << 4) - 1        |
                    |offset   = 1                   |
                    |                               |
                    |slots[XA_CHUNK_SIZE]           |
                    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                    |f|e|d|c|b|a|9|8|7|6|5|4|3|2|1|0|
                    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                       |                 |
                       |                 +--> 0xd15
                       +--> 0xd1e
```

从上面这个图我们可以看到

  * XA_CHUNK_SIZE表示了一个xa_node可以表示的比特位是多少
  * offset表示了自己在父节点上的位置
  * shift表示了自己在数字中表示第几段数字

总的来讲和系统的页表很像。有了这个图像概念在心里，有些函数就相对比较容易理解了。

不过在正式理解某些函数的工作细节之前，我们要看看代码中用来判断状态的辅助工作。因为不理解这些判断，让人觉得无法读懂代码。

# 眼花缭乱的状态判断

在xarray代码中，我们经常可以看到这样的代码

  * xa_is_internal()
  * xas_invalid()

然后你就会想这都是在判断什么情况？每次看到这些判断，我都要跳转到定义，然后想一下这次符合的是什么状态，又避免了哪种状态。如果另一个状态发生，又会是什么情况。所以很耗时，经常打断代码阅读，而且完全没有对整个数据结构有全面的理解。

现在是时候好好总结一下了。

## entry的种类

代码中有好些个地方对作者如何编排entry有描述，不过没有集中，所以有时候看起来会有些吃力。我们在这里汇总一下：

xarray.h的开头

```
/*
 * The bottom two bits of the entry determine how the XArray interprets
 * the contents:
 *
 * 00: Pointer entry
 * 10: Internal entry
 * x1: Value entry or tagged pointer
 *
 * Attempting to store internal entries in the XArray is a bug.
 *
 * Most internal entries are pointers to the next node in the tree.
 * The following internal entries have a special meaning:
 *
 * 0-62: Sibling entries
 * 256: Retry entry
 * 257: Zero entry
 *
 * Errors are also represented as internal entries, but use the negative
 * space (-4094 to -2).  They're never stored in the slots array; only
 * returned by the normal API.
 */
```

xa_mk_internal()的注释
```
* Internal entries are used for a number of purposes.  Entries 0-255 are
* used for sibling entries (only 0-62 are used by the current code).  256
* is used for the retry entry.  257 is used for the reserved / zero entry.
* Negative internal entries are used to represent errnos.  Node pointers
* are also tagged as internal entries in some situations.
```

还有一个地方就是xa_dump_entry()代码本身

根据这些注释，我们可以把entry分成这么集中情况：

```
(00)Pointer:
(x1)Value:
(10)Internal:
   [0, 255]       sibling entries       --+- advanced entry
   256            retry entry           -/
   257            zero entry
   > 4096         node entry(without left shift)
   >= -MAX_ERRNO  Error(never stored in slots array)
```

嗯，我相信到这里你会说这不就是把注释的东西拷贝了一份么？是的，你说的对。不过我请大家注意两个地方：

  * node entry在编码时，没有左移。是直接+2得到的。这是假设了我们的node指针低位为0
  * 错误状态不会存放在slots中。你仔细看node的定义其实包含了error，那这么判断不会出错么？因为error永远不会出现在slots中，只能存放在xas->xa_node。这样就重复利用了一段编码。

正是因为如此，代码中有两套判断状态的辅助函数

  * 判断entry的 xa_is_xxx()
  * 判断xas的   xas_xxx()

## 判断entry

这部分的函数判断一个entry的类型。

```
xa_is_value():    ((unsigned long)entry & 1)
xa_is_internal(): ((unsigned long)entry & 3) == 2
xa_is_sibling():  xa_is_internal() && entry < xa_mk_internal(XA_CHUNK_SIZE-1)
xa_is_retry():    xa_mk_internal(256)
xa_is_zero():     xa_mk_internal(257)
xa_is_node():     xa_is_internal() && > 4096, which includes xa_is_err()
xa_is_err():      xa_is_internal() && entry >= xa_mk_internal(-MAX_ERRNO)
                  the entry here never stored in slots array
```

这个判断的顺序和上一节展示的entry种类正好匹配。

## 判断xas

更确切的说，这部分函数判断的是xas->xa_node。在xarray的操作中，作者利用xas->xa_node保存操作过程中的返回值和状态。

在列出判断xas状态函数前，我们先来看一下三个新的定义：

```
#define XA_ERROR(errno) ((struct xa_node *)(((unsigned long)errno << 2) | 2UL))
#define XAS_BOUNDS	((struct xa_node *)1UL)
#define XAS_RESTART	((struct xa_node *)3UL)
```

这些是会保存在xas->xa_node中的特殊值，用来判断当前xas的状态。而判断xas状态的函数也由此展开

```
xas_invalid(): (unsigned long)xas->xa_node & 3
xas_frozen():  node & 2
xas_top():     node <= XAS_RESTART
xas_is_node(): xas_valid(xas) && xas->xa_node => !((unsigned long)xas->xa_node & 3)
               this only check xas->xa_node, which would be set to:
               * NULL(0)
               * XAS_BOUNDS(1)/XAS_RESTART(3)
               * XA_ERROR()
               * xa_to_node(entry)
xas_error():   xa_err(xas->xa_node)
```

其中我们可以看到，xas->xa_node上赋值的只有这么几种，而不是所有entry类型。
有了这些，在后面代码阅读中，我们将事半功倍。

# 代码分析

此时，我想我们已经准备好了真正理解代码的奥妙了。

## xas_create

在开始的图中，我们看到对每一个index，我们有一个slot对应。而这个slot中，则存放了我们想要存放的内容。

然而这个slot不是凭空出现的，就好像我们先要造好房子才能住进去。这个建造好slot的过程由xas_create函数来完成。

这个函数可以分成两个部分：

  * xas_expand： 扶摇直上
  * xas_descend：开枝散叶

整个xarray能表示的index范围由这个xarray的层级限制，就好像一栋房子能有多少户房间受到房子楼层的限制。在盖起来之前先要把地基给打好。这个就是xas_expand做的工作。

理论上，打好地基我们可以把整栋房子都盖起来。但是在软件世界，我们为了节约时间，节约成本，我们就只盖起你需要的那一间。这部分的工作由xas_descend来完成。

## xas_store

房子盖好了，有了合适的空间，我们就该把想要保存的东西放到指定的空间了。这个工作就交给了xas_store，也可以看到为了确保有足够的空间，xas_store调用了xas_create。

简单来说，xas_store的工作就是找到slot，并将entry赋值给slot。然而实际的代码要复杂得多，原因是xarray还支持一种“指数”存储的方法。

## xa_store_order

这个函数就是实现“指数”存储的方法，说来也简单，就是套了XA_STATE_ORDER的xas_store。既然这是重要差别，那我们就来看看究竟做了什么。

### shift/sibs

```
#define __XA_STATE(array, index, shift, sibs)  {	\
   ...

#define XA_STATE_ORDER(name, array, index, order)		\
	struct xa_state name = __XA_STATE(array,		\
			(index >> order) << order,		\
			order - (order % XA_CHUNK_SHIFT),	\
			(1U << (order % XA_CHUNK_SHIFT)) - 1)
```

这个宏，传进去的是index 和 期望的order，然后将其转换为了xarray中的shift和sibs。

我们先来看看这些转换都做了什么：

  * (index >> order) << order: 这个是将index低位清零。
  * order - (order % XA_CHUNK_SHIFT): 这个找到order能表达的最大倍数的XA_CHUNK_SHIFT
  * (1U << (order % XA_CHUNK_SHIFT)) - 1: 表达了order中去除最大倍数XA_CHUNK_SHIFT后剩余的个数（减1）

是不是看上去有点摸不到头脑？ 不急，我们换个角度先看看shift/sibs的含义。

极端情况，order等于0时，也就是最普通存储单个值时，对应的shift和sibs都为0. 此时的含义为：

**当我们存储单个值，order为0时，我们是在shift为0的node上，占据了1=0+1个slot**

然后我们把这个含义扩展一下

  * shift代表我们存储的这个范围最后落在的node的shift
  * sibs代表这个范围在node上，会有多少兄弟

### 举个例子

有了这个含义在心里，我们再找个例子来验证一下

如： index = 0xd15, order = 2
经过转换后 **index = 0xd14, shift = 0, sibs = 3**

表示这个范围最后会落在一个shift为0的node上，且有3个兄弟。

那效果是不是这样呢？

```
xarray->xa_head = xa_node0
                  +-------------------------------+
                  |parent   = NULL                |
                  |shift    = 8                   |
                  |max_index= (1 << (8 + 4)) - 1  |
                  |offset                         |
                  |                               |
                  |slots[XA_CHUNK_SIZE]           |
                  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                  |f|e|d|c|b|a|9|8|7|6|5|4|3|2|1|0|
                  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                       |                         |
                       |                         |
                       v                         v
       xa_node2                                  xa_node1
       +-------------------------------+         +-------------------------------+
       |parent   = xa_node0            |         |parent   = xa_node0            |
       |shift    = 4                   |         |shift    = 4                   |
       |max_index= (1 << (4 + 4)) - 1  |         |max_index= (1 << (4 + 4)) - 1  |
       |offset   = d                   |         |offset   = 0                   |
       |                               |         |                               |
       |slots[XA_CHUNK_SIZE]           |         |slots[XA_CHUNK_SIZE]           |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+         +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |f|e|d|c|b|a|9|8|7|6|5|4|3|2|1|0|         |f|e|d|c|b|a|9|8|7|6|5|4|3|2|1|0|
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+         +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                                    |                         |
                                    |                         +--> [0x090, 0x09f]
                                    v
                    xa_node3
                    +-------------------------------+
                    |parent   = xa_node2            |
                    |shift    = 0                   |
                    |max_index= (1 << 4) - 1        |
                    |offset   = 1                   |
                    |                               |
                    |slots[XA_CHUNK_SIZE]           |
                    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                    |f|e|d|c|b|a|9|8|x|x|x|4|3|2|1|0|
                    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                       |                 |
                       |                 +--> 0xd15
                       +--> 0xd1e
```

也就是 [0xd14, 0xd17] 这一段会同时被设置为一个值。怎么样，和我们预想的一样把。

### 代码细节

好了，现在我们再来看xas_store中是如何处理存储带有order范围数值的。

```
offset = xas->xa_offset;
max = xas->xa_offset + xas->xa_sibs;
```

这里max就指定好了一段范围，当offset==max时，赋值的循环才会跳出。

```
entry = xa_mk_sibling(xas->xa_offset);
```

另外对于后续的slot，填入的值也有所变化。不是entry本身，而是表示自己是范围第一个值的兄弟。

现在我们再来看一个例子：

index = 0xd15, order = 4
经过转换后
index = 0xd10, shift = 4, sibs = 0

这个效果就是这样的

```
xarray->xa_head = xa_node0
                  +-------------------------------+
                  |parent   = NULL                |
                  |shift    = 8                   |
                  |max_index= (1 << (8 + 4)) - 1  |
                  |offset                         |
                  |                               |
                  |slots[XA_CHUNK_SIZE]           |
                  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                  |f|e|d|c|b|a|9|8|7|6|5|4|3|2|1|0|
                  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                       |                         |
                       |                         |
                       v                         v
       xa_node2                                  xa_node1
       +-------------------------------+         +-------------------------------+
       |parent   = xa_node0            |         |parent   = xa_node0            |
       |shift    = 4                   |         |shift    = 4                   |
       |max_index= (1 << (4 + 4)) - 1  |         |max_index= (1 << (4 + 4)) - 1  |
       |offset   = d                   |         |offset   = 0                   |
       |                               |         |                               |
       |slots[XA_CHUNK_SIZE]           |         |slots[XA_CHUNK_SIZE]           |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+         +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |f|e|d|c|b|a|9|8|7|6|5|4|3|2|1|0|         |f|e|d|c|b|a|9|8|7|6|5|4|3|2|1|0|
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+         +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                                    |                         |
                              [0xd10, 0xd1f]                  +--> [0x090, 0x09f]
```

因为order正好是一个node能存储的范围，所以这种情况下就不需要用一个xa_node来表示所有一样的内容。
也就是说转换过的xa_shift表示了存储时，应该在哪一层。xa_sibs表示了在这一层要占几个slot。（需加1）

我们再回过头来看, 和xa_shift相关的地方：

  * xas_create中，xas_descend过程中限制了 shift > order。也就是往下伸展的时候，只要到order的层级就够了，不需要再往下构造空间
  * xas_store中，xas->xa_shift < node->shift时， xas->xa_sibs就会清零

## xa_store_range/xas_set_range

比起按照order存入数据，更有意思的时xa_store_range--**将数据存入任意一段范围内**。

我们先来看一下整体的框架，有个全局的概念。

```
xa_store_range(first, last)
{
  do {
    xas_set_range(first, last)
    xas_store()
    first += xas_size()
  } while (first <= last)
}
```

而这里面的奥妙就在于函数xas_set_range，其目的就是将一个range最大限度按照order切分，然后把切分好的部分各自存储。

```
static void xas_set_range(struct xa_state *xas, unsigned long first,
		unsigned long last)
{
	unsigned int shift = 0;
	unsigned long sibs = last - first;
	unsigned int offset = XA_CHUNK_MASK;

	xas_set(xas, first);

	while ((first & XA_CHUNK_MASK) == 0) {
		if (sibs < XA_CHUNK_MASK)
			break;
		if ((sibs == XA_CHUNK_MASK) && (offset < XA_CHUNK_MASK))
			break;
		shift += XA_CHUNK_SHIFT;
		if (offset == XA_CHUNK_MASK)
			offset = sibs & XA_CHUNK_MASK;
		sibs >>= XA_CHUNK_SHIFT;
		first >>= XA_CHUNK_SHIFT;
	}

	offset = first & XA_CHUNK_MASK;
	if (offset + sibs > XA_CHUNK_MASK)
		sibs = XA_CHUNK_MASK - offset;
	if ((((first + sibs + 1) << shift) - 1) > last)
		sibs -= 1;

	xas->xa_shift = shift;
	xas->xa_sibs = sibs;
}
```

这个函数短短30行，我却看了好几天才算是略有领悟。为了避免下次再花这么长时间，在此记录一下理解。

  * 这个函数把[first, last]这个范围，按照shift(及其整数倍)做划分
  * 那个while循环只处理first是shift对齐的情况，如果某一个shift段的不是全0，就会跳出循环去设置
  * while循环中不断增加shift，用来找到更匹配的shift，也就是找到更大的可设置的order
  * 跳出循环有两种情况： 该[first, last]范围已能被当前的shift覆盖，或者last的低位不是全1[*]
  * 跳出循环后，offset比较容易确定，直接用first与上XA_CHUNK_MASK即可
  * 跳出循环后，sibs需要修正，也是有两种情况：sibs = last - first，如果不是迭代到了最后一层sibs肯定是超多XA_CHUNK_MASK的
    第二中情况是last的低位不是全1[*]

这么些解释中，有一个last为全1的情况比较特殊，需要进一步解释。

以XA_CHUNK_SHIFT为4举例。

对于范围[0, 0xff]，可以是一个shift=8的一个slot。
而对于[0, 0xfe]则是另外一个情况。首先shift就只能是4， 而能设置到的范围就变成了[0, 0xef]。
这有点像进位的意思。

# 特例

xarray的行为有些做了适配，和我最初设想的有出入。在这里举几个例子看看。

## 默认不改变当前order

比如在代码中按如下顺序执行：

```
xa_store_range(&xa, 2, 64, xa_mk_index(1), 0);
xa_store(&xa, 32, xa_mk_value(3), 0);
```

xa_store_range将[2, 64]分割成了， [0, 15] [16, 63] [64] 三个区间。

原先我会以为在store range后，xa_store会单独设置一个index，导致一个像空洞一样的状态。但实际上，结果是区间的形态没有变化。
还是[0, 15] [16, 63] [64]， 只是把[16, 63]这个区间的值改成了value(3)。

xas_store的函数调用过程简化如下：

```
xas_store
    xas_create
        xas_descend
    assign value
```

在descend过程中， 只会在entry为空的情况下在创建一层。而当我们遇到sibling的时候，会返回sibling的长兄，然后就退出了。

## order降级

上面的例子中我们看到，如果之前设置过一个order，那么后续的store还是会延续这个order。那如果我们真的需要改变其中一部分范围的值呢？

这里就要用上xas_split了。

```
       unsigned int old_order = 3;
       unsigned int new_order = 2;
       DEFINE_XARRAY(xa);
       XA_STATE_ORDER(xas, &xa, 4, new_order);

       xa_store_order(&xa, 0, old_order, xa_mk_value(5), 0);
       xa_dump(&xa);
       xas_split_alloc(&xas, xa_mk_value(3), old_order, 0);
       xas_split(&xas, xa_mk_value(3), old_order);
       xa_dump(&xa);
```

这个函数也蛮有意思的。

## 任意指针

之前我们在分析entry的种类时看到，对存入的指针要求低2位为0。因为这个符合了内核中分配出来的内存地址是4字节对齐的。

但这个世界总是有幺蛾子

```
commit 76b4e52995654af260f14558e0e07b5b039ae202
Author: Matthew Wilcox <willy@infradead.org>
Date:   Fri Dec 28 23:20:44 2018 -0500

    XArray: Permit storing 2-byte-aligned pointers

    On m68k, statically allocated pointers may only be two-byte aligned.
    This clashes with the XArray's method for tagging internal pointers.
    Permit storing these pointers in single slots (ie not in multislots).

    Signed-off-by: Matthew Wilcox <willy@infradead.org>
```

但是这么一增加，用来判断entry是否是node的函数xa_is_node就失效了。所以在函数 xas_load 和 xas_free_nodes中分别增加了判断。
如果已经是在最底层了，那么即便xa_is_node返回真，他也不是一个node。

当然这里其实还是有个限制，那就是我们不会把这个信息存储到高阶的node上。嗯，这真的不是一个很好的特例情况。

# 测试

xarray这个数据结构已经是比较复杂的了，所以内核中提供了对应的代码对这部分做测试用来保证代码质量。

而且还提供了用户态和内核态两种测试方式：

用户态：

```
sudo yum install userspace-rcu-devel.x86_64
cd tools/testing/radix-tree
make
./xarray
```

内核态：

```
先把测试模块配置上
TEST_XARRAY n -> m
make modules
insmod lib/test_xarray.ko
```
