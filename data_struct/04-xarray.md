现在我们来看看xarray这个数据结构。

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

其中我们可以看到，xas->xa_node上赋值的只有这么几种，不是所有entry类型。
有了这些，在后面代码阅读中，我们将事半功倍。
