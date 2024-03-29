本来以为对内核中内存的理解划分成两部分就算完结了

* [自底而上话内存][1]
* [虚拟内存空间][2]

随着学习的深入发现还要再划出一个分类 -- 回收再利用。毕竟系统中的内存是有限的，如何能够在有限的资源下服务好更多的进程就需要有**螺蛳壳里做道场**的精神。

linux内核的设计理念是尽量多使用内存，这样可以最大化系统性能。那什么时候通过什么手段开始回收内存的操作呢？

其判断标准就是[水线][4]也就是当空闲内存少于某个范围，那就开始回收部分内存，直到系统觉得内存足够应对用户的需求。

内核有两种触发内存回收的路径：

  * 主动：同步
  * 被动：异步

相当于不仅有人定期巡视，也有热线电话随叫随到。这部分牵扯到的调用路径比较多，我们可以用一张[Big Picture][8]来帮助我们理解之间的关系。并且为了更直观理解内存回收的过程，我们做个实验[手动触发回收][10]

真正到做回收的时候，我们就要考虑回收谁，留下谁。如何选取是一门很大的学问，目的是为了能够尽量减少回收对系统性能的影响。这里用到的就是[Page Frame Reclaim Algorithm][9]。

刚才我们看到，回收时对内存分成了**匿名页**和**文件页**。对于文件读取的页面，回收时可以放回文件中。而匿名页要释放再利用，就要靠[swapfile原理使用和演进][3]

内存回收这部分还是相对比较复杂，也有很多细节我不一定全部覆盖。下面是我之前搜到的一些参考文章，可以交叉学习。

[Linux中的内存回收1][5]

[Linux中的内存回收2][6]

[Linux中的内存调节之watermark][7]

[1]: /mm/00-memory_a_bottom_up_view.md
[2]: /virtual_mm/00-index.md
[3]: /mm_reclaim/01-swapfile.md
[4]: /mm_reclaim/02-watermark.md
[5]: https://zhuanlan.zhihu.com/p/70964195
[6]: https://zhuanlan.zhihu.com/p/72998605
[7]: https://zhuanlan.zhihu.com/p/73539328
[8]: /mm_reclaim/03-big_picture.md
[9]: /mm_reclaim/04-pfra.md
[10]: /mm_reclaim/05-do_reclaim.md
