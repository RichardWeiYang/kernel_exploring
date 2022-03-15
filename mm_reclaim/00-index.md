本来以为对内核中内存的理解划分成两部分就算完结了

* [自底而上话内存][1]
* [虚拟内存空间][2]

随着学习的深入发现还要再划出一个分类 -- 回收再利用。

毕竟系统中的内存是有限的，如何能够在有限的资源下服务好更多的进程就需要有**螺蛳壳里做道场**的精神。当然这部分的内容也极其繁杂，暂时只能一点点去完善和补齐。

先放上知乎上看到的内存回收的系列文章

[Linux中的内存回收1][5]

[Linux中的内存回收2][6]

[Linux中的内存调节之watermark][7]

linux内核的设计理念是尽量多使用内存，这样可以最大化系统性能。那什么时候通过什么手段开始回收内存的操作呢？那就是

[水线][4]

对于文件读取的页面，回收时可以放回文件中。而匿名页要释放再利用，就要靠

[swapfile原理使用和演进][3]

[Big Picture][8]

[1]: /mm/00-memory_a_bottom_up_view.md
[2]: /virtual_mm/00-index.md
[3]: /mm_reclaim/01-swapfile.md
[4]: /mm_reclaim/02-watermark.md
[5]: https://zhuanlan.zhihu.com/p/70964195
[6]: https://zhuanlan.zhihu.com/p/72998605
[7]: https://zhuanlan.zhihu.com/p/73539328
[8]: /mm_reclaim/03-big_picture.md
