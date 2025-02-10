反向映射是干什么用的呢？下面摘抄一段来自网络上的定义：

```
Reverse map is one of the feature in linux kernel, which with given a physical
page, can return a list of PTEs which point to that page.
```

当我们想要判断是否可以释放一块内存页时，这个工作就很有必要了。更多的细节可以参看[lwn][1].

这个功能经过了时间的演化才稳定了下来，大致来看经过了这么几个阶段：

    * Process scan
    * pte_chain
    * Object-based rmap with anon_vma
    * Improve anon_vma with anon_vma_chain
    * Replace same_anon_vma linked list with an interval tree
    * Reuse anon_vma

# Process scan

在[lwn][1]中记录到，最开始这个功能的实现是通过扫描系统中的所有进程页表来查找是否引用了某个特定的页。

虽然这个可以工作，但是这个工作实在是代价很大。接着社区就提出了新的方案。

# pte_chain

还是在[lwn][1]中提到，社区的第二个方案是在struct page中增加一个字段: pte_chain。也就是所有映射到本页的pte都会加入到这个链表。

这是一个出于直觉的想法，也确实解决了扫描的问题。只要遍历pte_chain就可以找到对应的页表了。

但是社区依然对这个方案表示了不满，因为增加一个字段在struct page中，将增加系统内存消耗。社区希望去除这么一个额外的内存消耗。

# Object-based rmap with anon_vma

接下来我们就开始进入“现代”了，相信在当前的内核代码中，依稀可以看到这个想法的影子。

首先我们来描述一下这个想法的背景，在[The object-based reverse-mapping][2]和[the return of objrmap][3]写到：

通常来说，用户的页面有两种类型：

    * Anonymous page
    * file-backed page

对于file-backed page，[The object-based reverse-mapping][2]提到我们已经有了一个**长**路径来找到对应的页表。这就意味这pte_chain对file-backed page是不必要的。

对anonymous pages就没有什么**长**路径了，[the return of objrmap][3]引入了anon_vma来解决这个问题。

虽然我没有找到具体是哪个commit引入的这个方法，但是我相信这一定是下一节提到的commit之前的情形。

在[the return of objrmap][3]有两个图来解释这个想法。

假如只有一个进程，那么整个数据结构从页面到页表的关系将如下图：

```
      page
      +--------------+<---------------------------+
      |              |                            |
      |mapping       |----+                       |
      +--------------+    |                       |
                          |                       |
                          |                       |      page table
      page                |                       |      +----------------+
      +--------------+<---|-----------------------|---+  |                |
      |              |    |                       |   |  |                |
      |mapping       |----+                       +---|--|-- pte1         |
      +--------------+    |                           |  |                |
                          |                       +---|--|-- pte2         |
                          |                       |   |  |                |
                          |                       |   +--|-- pte3         |
      page                |                       |      |                |
      +--------------+<---|-----------------------+      +----------------+
      |              |    |                                  ^
      |mapping       |----+                                  |
      +--------------+    |                                  |
                          |      vma                         |
                          +----->+--------------+------------+
                                 |              |
                                 |              |
                                 +--------------+
```

一个vma要包含多个页，所以有多个页的mapping指向同一个vma。加上**page->index**，这样就可以定位到对应的pte。

当进程进行fork后，情况就变得复杂了。下图做了一个解释：

```
       page
       +--------------+<--------------------------------+
       |              |                                 |
       |mapping       |                                 |
       +--------------+                                 |
             |                                          |     page table
             |                                     +----|---->+----------------+
             v                                    /     |     |                |
         anon_vma                               /       |     |                |
         +-----------+                        /         +-----|-- pte          |
         |head       |<--+                  /           |     |                |
         +-----------+   |                /             |     |                |
            ^            |              /               |     +----------------+
            |            |            /                 |
            |  vma       v          /                   |
            |  +--------------+   /                     |
            |  |anon_vma_node | /                       |     page table
            |  +--------------+           +-------------|---->+----------------+
            |            ^              /               |     |                |
            |            |            /                 |     |                |
            |  vma       v          /                   +-----|-- pte          |
            |  +--------------+   /                     |     |                |
            |  |anon_vma_node | /                       |     |                |
            |  +--------------+                         |     +----------------+
            |            ^                              |
            |            |                              |
            |  vma       v                              |     page table
            |  +--------------+-------------------------|---->+----------------+
            |  |anon_vma_node |                         |     |                |
            |  +--------------+                         |     |                |
            |            ^                              +-----|-- pte          |
            |            |                                    |                |
            |            |                                    |                |
            +------------+                                    +----------------+
```

进程fork后，匿名页将被映射到所有这些进程中，并且每个进程中都有一个vma对应到被映射的页。所以page->mapping就不可能同时指向进程树中的各自vma结构。此时anon_vma就出现了，用它来链接上进程树中所有包含该页面的vma，形成一个anon_vma链表。

所以在上图中，每个vma代表的是进程树中各自进程的同一块区域。

这部分的理解在这个版本前的代码中找到了印证：
commit 5beb49305251e5669852ed541e8e2f2f7696c53e.

```
    dup_mmap(mm, oldmm)
        tmp = kmem_cache_alloc()
        *tmp = *mpnt
        anon_vma_link(tmp)
            list_add_tail(&tmp->anon_vma_node, &tmp->anon_vma->head)
```

上图只是显示了这个版本在进程树上的一个侧面，另一个侧面是在同一个进程内，anon_vma也会包含同一个进程内**mergable**的vma。这部分的内容可以在函数anon_vma_prepare()找到印证。

到这里，好像一切都完美了。

# Improve anon_vma with anon_vma_chain

内核社区还想对上述版本进行优化。在[The case of the overly anonymous anon_vma][4]中说到，当进程不断fork，如达到1000个子进程每个进程有1000个匿名页时，这个anon_vma链表将会变得非常庞大。因为所有进程树公用同一个anon_vma。

社区提出的方案是：给每一个进程创建一个anon_vma，通过新的数据结构anon_vma_chain，把进程树中各自进程的anon_vma链接起来，而不是直接把vma链接起来。

这个想法的实现引入自：
commit 5beb49305251e5669852ed541e8e2f2f7696c53e
"mm: change anon_vma linking to fix multi-process server scalability issue"

这个实现要复杂得多，让我们一步步展开。

首先我们来简化一下将要遇到的一些术语：

  * av: anon_vma
  * avc: anon_vma_chain
  * vma: vm_area_struct

当进程第一个分配匿名页时，这个结构看上去像下面的图示：

```
                  same_anon_vma           same_vma   anon_vma_chain
             .......................      *************************
             .                     .      *                       *
     av      v                 avc v      v                vma    v
     +-----------+             +-------------+             +-------------+
     |           |<------------|anon_vma  vma|------------>|             |
     |           |             |             |             |             |
     +-----------+             +-------------+             +-------------+
             ^                     ^      ^                       ^
             .                     .      *                       *
             .......................      *************************
```

最初的版本中 . 和 * 是两个在avc上的链表：

   * same_anon_vma( . ): 属于同一个anon_vma的avc都链接在这里
   * same_vma( * )     : 对某个vma有映射的avc都链接在这里

目前版本中，save_vma还是链表，而same_anon_vma替换成了一个红黑树。原因因该是有了reuse后，一个anon_vma下会有多个vma，这样用红黑树去组织更利于查找到需要的vma。

所以当进程再分配一个属于另一个vma但在同一个anon_vma区域的匿名页，这个结构的状态如下：
PS: 这种情况发生在可以重用anon_vma的时候，见find_mergeable_anon_vma()。要求原来的vma和新的紧临，并且还没有被reuse过。

```
                  same_anon_vma           same_vma   anon_vma_chain
             .......................      *************************
             .                     .      *                       *
     av      v                 avc v      v                vma    v
     +-----------+             +-------------+             +-------------+
     |           |<------------|anon_vma  vma|------------>|             |
     |           |<----+       |             |             |             |
     +-----------+     |       +-------------+             +-------------+
             ^         |           ^      ^                       ^
             .         |           .      *                       *
             .         |           .      *************************
             .         |           .
             .         |           .      same_vma   anon_vma_chain
             .         |           .      *************************
             .         |           .      *                       *
             .         |       avc v      v                vma    v
             .         |       +-------------+             +-------------+
             .         +-------|anon_vma  vma|------------>|             |
             .                 |             |             |             |
             .                 +-------------+             +-------------+
             .                     ^      ^                       ^
             .                     .      *                       *
             .......................      *************************
```

链表还是那个链表，就是比之前胖了一点。

现在是时候看一下fork下的情况了。

```
             .......................      *************************
             .                     .      *                       *
     av      v                 avc v      v                vma    v
     +-----------+             +-------------+             +-------------+
 P   |           |<------------|anon_vma  vma|------------>|             |
     |           |<----+       |             |             |             |
     +-----------+      \      +-------------+             +-------------+
             ^                     ^      ^                       ^
             .           \         .      *                       *
             .                     .      *************************
             .            \        .
             .                     .
             .             \       .
             .                     .
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
     av      v                 avc v      v              \ vma    v
     +-----------+             +-------------+            >+-------------+
 C1  |           |<------------|anon_vma  vma|------------>|             |
     |           |             |             |             |             |
     +-----------+             +-------------+             +-------------+
             ^                     ^      ^                       ^
             .                     .      *                       *
             .......................      *************************
```

当进程P创建了子进程C1后，会拷贝一样的vma在子进程C1中。首先会创建这个vma自己的avc和anon_vma。这个过程和父进程是一样的。而不同的是此时还会创建一个avc，将子进程的vma链接到父进程的anon_vma上。这样以来，当我们寻找某个内存都映射到哪些进程中时，就可以通过父进程的anon_vma上same_anon_vma来找到子进程也映射了某个页面。

接着我们挑战一下有两个fork的时的情形。

```
             .......................      *************************
             .                     .      *                       *
     av      v                 avc v      v                vma    v
     +-----------+             +-------------+             +-------------+
 C2  |           |<------------|anon_vma  vma|------------>|             |
     |           |             |             |             |             |
     +-----------+             +-------------+           ->+-------------+
             ^                     ^      ^             /         ^
             .                     .      *                       *
             .......................      *           /           *
                                          *                       *
                                          *         /             *
             .......................      *                       *
             .                     .      *       /               *
             .                     v      v                       *
             .                 +-------------+  /                 *
             .                 |             |                    *
             .                /|anon_vma  vma|/                   *
             .                 +-------------+                    *
             .               /     ^      ^                       *
             .                     .      *                       *
             .              /      .      *************************
             .                     .
             .             /       .
             .                     .      ************************
             .            /        .      *                       *
     av      v                 avc v      v                vma    v
     +-----------+<------/     +-------------+             +-------------+
 P   |           |<------------|anon_vma  vma|------------>|             |
     |           |<------      |             |             |             |
     +-----------+       \     +-------------+             +-------------+
             ^                     ^      ^                       ^
             .            \        .      *                       *
             .                     .      *************************
             .             \       .
             .                     .
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
             .......................      *           \           *
             .                     .      *                       *
     av      v                 avc v      v             \  vma    v
     +-----------+             +-------------+           ->+-------------+
 C1  |           |<------------|anon_vma  vma|------------>|             |
     |           |             |             |             |             |
     +-----------+             +-------------+             +-------------+
             ^                     ^      ^                       ^
             .                     .      *                       *
             .......................      *************************
```

上图显示了对称的美感，当然更重要的是显示了一个匿名页如何通过anon_vma->same_anon_vma来找到所有映射到自己的进程。不过问题是，这么一个设计对之前的实现来讲究竟改进在哪里？

我认为改进发生在COW之后。比如，如果C1的匿名页发生了COW，新的匿名页page->mapping就会指向C1的anon_vma而不是P的。这样当我们想要查询这个新的匿名页的映射情况，就不需要从根上找了。

接着再进一步，我很好奇当子进程发生fork时的情形。

```
             .......................      *************************
             .                     .      *                       *
     av      v                 avc v      v                vma    v
     +-----------+             +-------------+             +-------------+
 P   |           |<------------|anon_vma  vma|------------>|     anon_vma| -> points to P's av
     |           |<----+       |             |             |             |
     +-----------+     |\      +-------------+             +-------------+
             ^         |           ^      ^                       ^
             .         | \         .      *                       *
             .         |           .      *************************
             .         |  \        .
             .         |           .
             .         |   \       .                        vma
             .         |           .      *****************>+-------------+<**************************
             .         |    \      .      *                 |     anon_vma| -> points to C's av      *
             .         |           .      *              -->|             |<---------+               *
             .         |     \ avc v      v             /   +-------------+          |               *
             .         |       +-------------+         /                             |               *
             .         |      \|anon_vma  vma|---------                              |               *
             .         |       |             |                                       |               *
             .         |       +-------------+<*********************************     |               *
             .         |           ^                                           *     |               *
             .         |           .                                           *     |               *
             .         |           .                   ......................  *     |               *
             .         |           .                   .                    .  *     |               *
             .         |           .        av         v                avc v  v     |               *
             .         |           .        +-------------+             +-------------+              *
             .         |           .    C   |             |<------------|anon_vma  vma|<**************
             .         |           .        |             |<---+        |             |
             .         |           .        +-------------+    |        +-------------+
             .         |           .                   ^       |              ^
             .         |           .                   .       |              .
             .         |           .                   .   avc |              .
             .         |           .                   .   +---------------+  .
             .         |           .                   ...>|anon_vma       |<..
             .         |           .                       |               |
             .         |           .      *****************|            vma|****
             .         |           .      *                +---------------+   *
             .         |           .      *                              |     *
             .         |       avc v      v                              |     *
             .         |       +-------------+                           |     *
             .         +-------|anon_vma  vma|\                          |     *
             .                 |             |  \                        |     *
             .                 +-------------+    \                      |     *
             .                     ^      ^         \                    |     *
             .                     .      *           \                  |     *
             .......................      *             \                |     *
                                          *               \              |     *
                                          *                 \            |     *
                                          *                   \          |     *
             .......................      *                     \        |     *
             .                     .      *                       v      |     *
     av      v                 avc v      v                         vma  v     v
     +-----------+             +-------------+                      +-------------+
 GC  |           |<------------|anon_vma  vma|--------------------->|     anon_vma| -> points to GC's av
     |           |             |             |                      |             |
     +-----------+             +-------------+                      +-------------+
             ^                     ^      ^                                    ^
             .                     .      *                                    *
             .......................      **************************************
```

这张图显示了进程P创建了C，接着C又创建了GC。其中有几点很有意思的地方：

    * 不论是多深的子进程，都会给子进程的vma创建一个avc链接到根进程的same_anon_vma上
    * 对一个多层次的子进程，vma会链接进所有父进程的anon_vma。所以GC进程中vma->same_vma链表上有三个节点。两个父辈，一个自己。

## Page -> vma

经过了这么多的重构，看上去我们已经离我们的初始目标有点远了。

让我们看一下我们真正想要达到的目的：

    given a physical page, can return a list of PTEs which point to that page

在这次重构前，事情还是很清楚的。对于整个进程树中相关的匿名页只有一个anon_vma。

这一点依然没有变，函数do_anonymous_page()和anon_vma_fork()显示了这点：

  * do_anonymous_page() will get the proper anon_vma and set it to
    page->mapping.
  * anon_vma_fork() will create new anon_vma but page->mapping is not changed

区别发生在COW情况下，而这种情况就是这个改动想要提升的。

没有改动之前，page->mapping依然指向唯一的anon_vma。在commit 5beb49305251改动之后，会为子进程创建自己的anon_vma。在函数anon_vma_fork和do_wp_page可以看到区别。

  * anon_vma_fork() create dedicate anon_vma for this vma and assign this
    anon_vma to vma->anon_vma
  * do_wp_page() will call page_add_new_anon_rmap() to set new_page->mapping
    to this dedicate anon_vma

此时我不禁又产生了一个问题。当一个vma中的所有匿名页都发生了COW，那么这个vma就没有必要继续留在父进程的same_anon_vma链表中了。那么我们会把它从链表中断开么？

# Replace same_anon_vma linked list with an interval tree

这是一个简单的改进。想法很直观，就是把same_anon_vma链表转换成interval tree(红黑树)。

这个改动在此引入 commit bf181b9f9d8dfbba58b23441ad60d0bc33806d64
"mm anon rmap: replace same_anon_vma linked list with an interval tree"

# Reuse anon_vma

如果进程不断的fork，上图中显示的anon_vma树将非常庞大。社区又提出了改进方法。

具体的代码在这个提交：

commit 7a3ef208e662f4b63d43a23f61a64a129c525bbc
"mm: prevent endless growth of anon_vma hierarchy"

先来看看提交记录：

```
    Constantly forking task causes unlimited grow of anon_vma chain.  Each
    next child allocates new level of anon_vmas and links vma to all
    previous levels because pages might be inherited from any level.

    This patch adds heuristic which decides to reuse existing anon_vma
    instead of forking new one.  It adds counter anon_vma->degree which
    counts linked vmas and directly descending anon_vmas and reuses anon_vma
    if counter is lower than two.  As a result each anon_vma has either vma
    or at least two descending anon_vmas.  In such trees half of nodes are
    leafs with alive vmas, thus count of anon_vmas is no more than two times
    bigger than count of vmas.

    This heuristic reuses anon_vmas as few as possible because each reuse
    adds false aliasing among vmas and rmap walker ought to scan more ptes
    when it searches where page is might be mapped.
```

这个patch的东西不多，但着实让我花了点时间去理解。下面我列举一些我的理解。

```
@@ -188,6 +190,8 @@ int anon_vma_prepare(struct vm_area_struct *vma)
                if (likely(!vma->anon_vma)) {
                        vma->anon_vma = anon_vma;
                        anon_vma_chain_link(vma, avc, anon_vma);
+                       /* vma reference or self-parent link for new root */
+                       anon_vma->degree++;
                        allocated = NULL;
                        avc = NULL;
```

当我看到这里还要加1时非常困惑，因为在anon_vma_alloc()中degree已经设置成了1。在注释的帮助下，我才理解degree代表了三种值：

    * VMAs which points to this anon_vma.
    * child of anon_vmas
    * LBNL, self-parent root

最初我混淆了第三种用途。

[1]: http://lwn.net/2002/0124/kernel.php3
[2]: https://lwn.net/Articles/23732/
[3]: https://lwn.net/Articles/75198/
[4]: https://lwn.net/Articles/383162/
