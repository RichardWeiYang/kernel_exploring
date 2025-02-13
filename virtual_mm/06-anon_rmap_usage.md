反向映射的目的是为了找到所有映射到某一个页面的页表项，从而可以对目标页做一些操作，比如切断映射。

反向映射一直是一个非常神奇的存在，今天我们就好好探索一下这个知识点。

# 创建

在反向匿名映射中出了page struct，一共有三个相关的数据结构：

  * vm_area_struct
  * anon_vma
  * anon_vma_chain

第一个数据结构我们已经见过了，是一个老朋友。而后两者就是为了构造反向匿名映射而新生的。我们先来看看这两个新的数据结构的样子。

## anon_vma

```
    anon_vma
    +----------------------------+
    |root                        |  = self
    |parent                      |  = self
    |    (struct anon_vma*)      |
    |refcount                    |  = 1
    |    (atomic_t)              |
    |num_children                |  = 0
    |num_active_vmas             |  = 0
    |    (unsigned long)         |
    |rb_root                     |
    |    (struct rb_root_cached) |
    +----------------------------+
```

这个结构由anon_vma_alloc()函数统一生成，上图中也显示了创造出来时候的样子。从这里看，也就是个带有上下级关系的这么一个结构。

## anon_vma_chain

```
    anon_vma_chain
    +----------------------------+
    |vma                         |
    |    (struct vm_area_struct*)|
    |anon_vma                    |
    |    (struct anon_vma*)      |
    |                            |
    |rb                          |
    |    (struct rb_node)        |
    |same_vma                    |
    |    (struct list_head)      |
    +----------------------------+
```

这个结构由anon_vma_chain_alloc()统一创建，貌似创建完了也不需要初始化，在anon_vma_chain_link()中直接都被赋值了。

## 组合

到这里，大家应该感觉怪怪的，都不知道这些东西是个啥。别急，我把这些东西组合起来，可能你就会有一些感觉了。

```
    +----------------------------------------------------------------------+
    |                                                                      |
    |       rb_root              rb      same_vma   anon_vma_chain         |
    |       .......................      *************************         |
    v       .                     .      *                       *         |
    av      v                 avc v      v                vma    v         |
    +-----------+             +-------------+             +-------------+  |
    |           |             |             |             |             |  |
    |           |<------------|anon_vma  vma|------------>|     anon_vma|--+
    +-----------+             +-------------+             +-------------+
            ^                     ^      ^                       ^
            .                     .      *                       *
            .......................      *************************
```

在这里，我们把这三个重要的数据结构之间的组合关系展现给大家。当然这只是最简单的组合关系，目的是为了让大家能有一个感性的认识。

  * anon_vma_chain链接了anon_vma和vma
  * vma则会有指针指向自己的anon_vma

空口无凭，眼见为实。那为什么会长成这样的呢？ 接下来我们就来看看在内核中我们是如何将这些数据结构链接起来的。

# 链接

上一节的最后，我们看到了三个重要的数据结构通过链表和树连接在了一起，这一节我们就来看看他们是怎么连接起来的。

## anon_vma_chain_link()

往简单了讲，要连接这三个重要的数据结构，都靠一个函数： anon_vma_chain_link(vma, avc, anon_vma)。而这个函数本身简单到令人发指，以至于我能把整个定义给大家展示出来。

```
    static void anon_vma_chain_link(struct vm_area_struct *vma,
    				struct anon_vma_chain *avc,
    				struct anon_vma *anon_vma)
    {
    	avc->vma = vma;
    	avc->anon_vma = anon_vma;
    	list_add(&avc->same_vma, &vma->anon_vma_chain);
    	anon_vma_interval_tree_insert(avc, &anon_vma->rb_root);
    }
```

你对照这上面的图一看，和图上显示的一摸一样没有任何多余的步骤。

但是关键的来了，如果你以为一切就这这么简单，那就too young too simple了啊。

接下来我们将从anon_vma_chain_link()函数被调用的关系入手，去看看在实际运行中究竟会演化出什么样的变化来。

## do_anonymous_page()

首先出场的是函数do_anonymous_page，这个函数是在匿名页缺页中断时会调用的函数。

```
    do_anonymous_page(vmf)
        __anon_vma_prepare(vma)
            avc = anon_vma_chain_alloc()
            anon_vma = find_mergeable_anon_vma(vma)
            anon_vma = anon_vma_alloc()
            vma->anon_vma = anon_vma
            anon_vma_chain_link(vma, avc, anon_vma)
```

从上面的流程可以看出，当发生缺页中断时，内核会给对应的vma构造anon_vma，并且利用avc去链接这两者。这种可以说是系统中最简单的例子，也是上图中显示的情况。

细心的人可能已经看到了，上面有一种情况是find_mergeable_anon_vma。如果这个函数返回一个可以重用的anon_vma，那么内核就可以利用原有的anon_vma了。此时这个图我们可以画成这样。

```
                  same_anon_vma           same_vma   anon_vma_chain
             .......................      *************************
             .                     .      *                       *
     av      v                 avc v      v                vma    v
     +-----------+             +-------------+             +-------------+
     |           |<------------|anon_vma  vma|------------>|     anon_vma|--->av
     |           |<----+       |             |             |             |
     +-----------+     |       +-------------+             +-------------+
             ^         |           ^      ^                       ^
             .         |           .      *                       *
             .         |           .      *************************
             .         |           .
             .         |           .      same_vma   anon_vma_chain
             .         |           .      *************************
             .         |           .      *                       *
             .         |       avc v      v                vma2   v
             .         |       +-------------+             +-------------+
             .         +-------|anon_vma  vma|------------>|     anon_vma|--->av
             .                 |             |             |             |
             .                 +-------------+             +-------------+
             .                     ^      ^                       ^
             .                     .      *                       *
             .......................      *************************
```

其实此处我画得不够精确，av 和 avc之间应当是树的关系，而不是现在显示的链表的关系。但是我想意思已经表达清楚，即在一个进程中多个vma可以共享同一个anon_vma作为匿名映射的节点。

## anon_vma_fork()

看过了在单个进程中的情况，接下来我们来看看创建一个子进程时如何调整这个数据结构。这个过程由anon_vma_fork处理。

```
    anon_vma_fork(vma, pvma)
        anon_vma_clone(vma, pvma)
        anon_vma = anon_vma_alloc()
        avc = anon_vma_chain_alloc()
        anon_vma->root = pvma->anon_vma->root
        anon_vma->parent = pvma->anon_vma
        vma->anon_vma = anon_vma
        anon_vma_chain_link(vma, avc, anon_vma)
```

这个函数很有意思，我还真是花了些时间去理解它。最开始有点看不清，所以我干脆退回到最简单的状态，也就是当前进程是根进程的时候。此时我才大致的了解了一点fork时究竟发生了什么。

话不多说，还是用一个图来表达

```
                  same_anon_vma           same_vma   anon_vma_chain
             .......................      *************************
             .                     .      *                       *
     av      v                 avc v      v                src    v
     +-----------+             +-------------+             +-------------+
 P   |           |<------------|anon_vma  vma|------------>|     anon_vma|--->av
     |           |<----+       |             |             |             |
     +-----------+      \      +-------------+             +-------------+
             ^                     ^      ^                       ^
             .           \         .      *                       *
             .                     .      *************************
             .            \        .
             .                     .
             .             \       .
             .                     .      same_vma   anon_vma_chain
             .              \      .      *************************
             .                     .      *                       *
             .               \ avc v      v                       *
             .                 +-------------+                    *
             .                \|anon_vma  vma|\                   *
             .                 |             |                    *
             .                 +-------------+  \                 *
             .                    ^       ^                       *
             .                    .       *       \               *
             ......................       *                       *
                                          *         \             *
                                          *                       *
                                          *           \           *
             .......................      *                       *
             .                     .      *             \         *
     c_av    v                 avc v      v              \ dst    v
     +-----------+             +-------------+            >+-------------+
 C1  |           |<------------|anon_vma  vma|------------>|     anon_vma|--->c_av
     |           |             |             |             |             |
     +-----------+             +-------------+             +-------------+
             ^                     ^      ^                       ^
             .                     .      *                       *
             .......................      *************************
```

P是父进程，C1是他的一个子进程。当发生fork时，page->mapping没有发生改变，所以依然需要能够从父进程的anon_vma上搜索到对应的页表。此时就得在父进程的rb_root树中保留一个子进程的avc。同时子进程又拥有自己的一套anon_vma。

可以说这个真的是非常有意思的。

对了，代码中还有一个函数anon_vma_clone，在这里我就不展开了。留给大家下来思考一下下。

## anon_vma之间的层级关系

在struct anon_vma的定义里我们看到有两个成员root/parent。那这会是形成怎么样的关系呢？

此外结构中还有一个成员num_children，这个是怎么变化的呢？我们来一步步看过来。

首先我们看只有自己的情况， __anon_vma_prepare()并且没有找到mergeable的。

```
                     av
                     +-------------+<---+
                 P   |root         |    |
                     |parent    ---|----+
                     |             |
                     |num_children | = 1
                     +-------------+
```

此时root/parent都指向自己，anon_vma_alloc()初始化如此。并且num_children增加到1。

然后我们假设这个父进程fork了两个子进程, anon_vma_fork()且anon_vma_clone()没有重用的情况。

```
                     av
                     +-------------+<---+
                 P   |root         |    |
                     |parent    ---|----+
                     |             |
                     |num_children | = 3
                     +-------------+
                              ^
                              |
          av                  |         av
          +-------------+     |         +-------------+
      C1  |             |     |     C2  |             |
          |root      ---|-----+---------|root         |
          |parent    ---|-----+---------|parent       |
          |             |               |             |
          |num_children | = 0           |num_children | = 0
          +-------------+               +-------------+
```

此时root/parent都指向父进程中的av，且父进程中的num_children增加到3，子进程中的num_children则保持为0。


接下来我们看看，子进程C2也发生fork后的情况。同样，这个过程也发生在anon_vma_fork中。

```
                     av
                     +-------------+<---+---------------------------+
                 P   |root         |    |                           |
                     |parent    ---|----+                           |
                     |             |                                |
                     |num_children | = 3                            |
                     +-------------+                                |
                              ^                                     |
                              |                                     |
          av                  |         av                          |
          +-------------+     |         +-------------+<-------+    |
      C1  |             |     |     C2  |             |        |    |
          |root      ---|-----+---------|root         |        |    |
          |parent    ---|-----+---------|parent       |        |    |
          |             |               |             |        |    |
          |num_children | = 0           |num_children | = 1    |    |
          +-------------+               +-------------+        |    |
                                                               |    |
                                                               |    |
                                        av                     |    |
                                        +-------------+        |    |
                                    GC  |             |        |    |
                                        |parent    ---|--------+    |
                                        |root      ---|-------------+
                                        |             |
                                        |num_children | = 0
                                        +-------------+
```

这时候就看出root/parent之间的区别了，parent指向上一级，而root指向根。

# 使用

好了，到了这里我们已经拥有了一个非常强悍的武器 -- 匿名反向映射。有了他我们就可以指哪打哪了。

内核也已经给我们准备好了扣动这个核武器的板机 -- rmap_walk_anon。

```
    rmap_walk_anon(page, rwc, true/false)
        anon_vma = page_anon_vma(page), get anon_vma from page->mapping
        pgoff_start = page_to_pgoff(page);
            return page_to_index(page)
        pgoff_end = pgoff_start + hpage_nr_pages(page) - 1;
        anon_vma_interval_tree_foreach(avc, &anon_vma->rb_root, pgoff_start, pgoff_end)
        rwc->rmap_one(page, vma, address, rwc->arg) -> do the real work
```

有了上面的基础知识，我想看这段代码就不难了。还记得上面看到过的那个rb_root么？对了，我们就是沿着这颗红黑树找到的vma，然后再找到了页表。

嗯，一切都感觉这么的完美。
