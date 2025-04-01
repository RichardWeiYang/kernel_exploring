反向映射的目的是为了找到所有映射到某一个页面的页表项，从而可以对目标页做一些操作，比如切断映射。

反向映射一直是一个非常神奇的存在，今天我们就好好探索一下这个知识点。

# 相关数据结构

在反向匿名映射中除了page struct，一共有三个相关的数据结构：

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

# 三种链接anon_vma/vma的情况

上一节的最后，我们看到了三个重要的数据结构通过链表和树连接在了一起，这一节我们就来看看他们是怎么连接起来的。

链接的核心函数是 anon_vma_chain_link()

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

## __anon_vma_prepare()

首先出场的是函数__anon_vma_prepare().

```
   __anon_vma_prepare(vma)
       avc = anon_vma_chain_alloc()
       anon_vma = find_mergeable_anon_vma(vma)      // 先找mergeable的
       anon_vma = anon_vma_alloc()                  // 找不到再分配
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

## anon_vma_clone()

第二个出场的是函数anon_vma_clone().

```
   anon_vma_clone(dst, src)
       avc = anon_vma_chain_alloc()
       anon_vma = pavc->anon_vma
       anon_vma_chain_link(dst, avc, anon_vma)
```

这个函数的作用是将dst和父进程中的src的anon_vma链接起来。具体的图解在下一节看。

## anon_vma_fork()

看过了在单个进程中的情况，接下来我们来看看创建一个子进程时如何调整这个数据结构。这个过程由anon_vma_fork处理。

```
    anon_vma_fork(vma, pvma)
        anon_vma_clone(vma, pvma)
        if (vma->anon_vma)                            // 如果clone中已经复用了父亲的anon_vma，直接返回
            return 0;
        anon_vma = anon_vma_alloc()
        avc = anon_vma_chain_alloc()
        anon_vma->root = pvma->anon_vma->root
        anon_vma->parent = pvma->anon_vma
        vma->anon_vma = anon_vma
        anon_vma_chain_link(vma, avc, anon_vma)       // 链接新增的子进程自己的anon_vma
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

P是父进程，C1是他的一个子进程。当发生fork时，page->mapping没有发生改变，所以依然需要能够从父进程的anon_vma上搜索到对应的页表。此时就得在父进程的rb_root树中保留一个子进程的avc。同时子进程又拥有自己的一套anon_vma，这样发生cow后，新的page->mapping就可以指向子进程，而不用再从父进程开始找了。

可以说这个真的是非常有意思的。

# 层级关系

## anon_vma之间的层级关系

在struct anon_vma的定义里我们看到有两个成员root/parent。那这会是形成怎么样的关系呢？

此外结构中还有两个成员num_children/num_active_vmas，这个是怎么变化的呢？我们来一步步看过来。

首先我们看只有自己的情况， __anon_vma_prepare()并且没有找到mergeable的。

```
                     av
                     +-----------------+<---+
                 P   |root             |    |
                     |parent        ---|----+
                     |                 |
                     |refcount         | = 1
                     |num_children     | = 1
                     |num_active_vmas  | = 1
                     +-----------------+
```

此时root/parent都指向自己，anon_vma_alloc()初始化如此。并且num_children增加到1。

如果在__anon_vma_prepare()发生了重用，则会是这样。

```
                     av
                     +-----------------+<---+
                 P   |root             |    |
                     |parent        ---|----+
                     |                 |
                     |refcount         | = 1
                     |num_children     | = 1
                     |num_active_vmas  | = 2
                     +-----------------+
```

也就是当前这个anon_vma被写到两个vma->anon_vma中。

然后我们假设这个父进程fork了两个子进程, anon_vma_fork()且anon_vma_clone()没有重用的情况。

```
                         av
                         +-----------------+<---+
                     P   |root             |    |
                         |parent        ---|----+
                         |                 |
                         |refcount         | = 3
                         |num_children     | = 3
                         |num_active_vmas  | = 1
                         +-----------------+
                                  ^
                                  |
          av                      |         av
          +-----------------+     |         +-----------------+
      C1  |                 |     |     C2  |                 |
          |root          ---|-----+---------|root             |
          |parent        ---|-----+---------|parent           |
          |                 |               |                 |
          |refcount         | = 1           |refcount         | = 1
          |num_children     | = 0           |num_children     | = 0
          |num_active_vmas  | = 1           |num_active_vmas  | = 1
          +-----------------+               +-----------------+
```

此时root/parent都指向父进程中的av，且父进程中的num_children增加到3，子进程中的num_children则保持为0。


接下来我们看看，子进程C2也发生fork后的情况。同样，这个过程也发生在anon_vma_fork，且没有重用时。

```
                         av
                         +-----------------+<---+---------------------------+
                     P   |root             |    |                           |
                         |parent        ---|----+                           |
                         |                 |                                |
                         |refcount         | = 4                            |
                         |num_children     | = 3                            |
                         |num_active_vmas  | = 1                            |
                         +-----------------+                                |
                                  ^                                         |
                                  |                                         |
          av                      |         av                              |
          +-----------------+     |         +-----------------+<-------+    |
      C1  |                 |     |     C2  |                 |        |    |
          |root          ---|-----+---------|root             |        |    |
          |parent        ---|-----+---------|parent           |        |    |
          |                 |               |                 |        |    |
          |refcount         | = 1           |refcount         | = 1    |    |
          |num_children     | = 0           |num_children     | = 1    |    |
          |num_active_vmas  | = 1           |num_active_vmas  | = 1    |    |
          +-----------------+               +-----------------+        |    |
                                                                       |    |
                                                                       |    |
                                            av                         |    |
                                            +-----------------+        |    |
                                        GC  |                 |        |    |
                                            |parent        ---|--------+    |
                                            |root          ---|-------------+
                                            |                 |
                                            |refcount         | = 1
                                            |num_children     | = 0
                                            |num_active_vmas  | = 1
                                            +-----------------+
```

这时候就看出root/parent之间的区别了，parent指向上一级，而root指向根。
而且num_children只用来表示下一层的数量，至于还有多少层就不管了。

## anon_vma的那些数字们

在上面的图中我们看到anon_vma上有三个动态变化的数字，上面已经介绍了各自的作用。这里再总结一下。

 * refcount: 整个树上的anon_vma的个数
 * num_children: 自己和自己直接孩子的个数
 * num_active_vmas: vma->anon_vma是自己的vma的个数

refcount保证了根anon_vma会最后被释放，只要还有一个子孙，自己就必须坚守在那！辛苦了。
num_children - 1 表示了当前还有多少直接的子进程

num_active_vmas就有点意思，有几种情况：

* 刚开始是1，表示有一个vma中的anon_vma指向自己
* 然后随着clone或者reuse，这个值会增加
* 但是如果父进程先与子进程退出了，这个值就变成0,说明没有人再指向自己。但是因为还有子进程，所以此时还必须坚守岗位。

## anon_vma上的interval tree

除了anon_vma之间有关联，rmap中非常重要的一个关系就是anon_vma和anon_vma_chain之间的联系。因为有这层关系的存在，才能找到一个page被哪些页表映射。

这是一个根在anon_vma的interval tree，而节点是avc。这棵树上连接了所有包含有page->mapping == anon_vma的vma。 (真有点绕。)

但是我觉得这棵树上其实没有很好利用interval tree的能力，因为这棵树上avc所指向的区域大多一致或者相近。

比如fork过程中，子进程的avc链接到父进程的树上。因为子进程和父进程的vma范围一直，所以这两个会挨着很近。这样以此类推，所有子进程都会在根上有一个节点，那父进程这棵树上其实有一堆一样范围的节点。

# 使用

## 连接page <-> anon_vma

这个过程是在发生page fault时，将page->mapping设置到事先分配好并保存在vma中的anon_vma。

比如说在匿名页中断过程中，连接两者的情况如下。其实所有的连接最后都由__folio_set_anon()来完成。

```
  do_anonymous_page()
    folio_add_new_anon_rmap(folio, vma, addr, RMAP_EXCLUSIVE)
      __folio_set_anon(folio, vma, address, exclusive)
        anon_vma = vma->anon_vma
	WRITE_ONCE(folio->mapping, (struct address_space *)anon_vma)
```

## 解开page <-> anon_vma

对应连接，就有解开。 恩，为啥会要解开？

但是有个函数叫__folio_remove_rmap()，我理解一个page加好了mapping后应该就不会变了。不知道这个函数是干啥的。

## 通过anon_vma找到page被映射到的所有进程

好了，到了这里我们已经拥有了一个非常强悍的武器 -- 匿名反向映射。有了他我们就可以指哪打哪了。

内核也已经给我们准备好了扣动这个核武器的板机 -- rmap_walk_anon。

```
    rmap_walk_anon(page, rwc, true/false)
        anon_vma = page_anon_vma(page), get anon_vma from page->mapping
        pgoff_start = page_to_pgoff(page);
            return page_to_index(page)
        pgoff_end = pgoff_start + hpage_nr_pages(page) - 1;

        // 在interval tree中找pgoff_start/pgoff_end之间的avc
        anon_vma_interval_tree_foreach(avc, &anon_vma->rb_root, pgoff_start, pgoff_end)
          // 接下来就是 try_to_unmap() 来干活了
          rwc->rmap_one(page, vma, address, rwc->arg) -> 
```

有了上面的基础知识，我想看这段代码就不难了。还记得上面看到过的那个rb_root么？对了，我们就是沿着这颗红黑树找到的vma，然后再找到了页表。

嗯，一切都感觉这么的完美。
