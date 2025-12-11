/proc/kpageflags保存了每个pfn对应的flags状态。

对应的内核处理函数在fs/proc/page.c，kpageflags_read。最后调用到stable_page_flags(page)。

大致扫了一眼，基本上是把flags里面的东西传到了用户态。

这样的话就能在测试程序中判断对应的page是否符合预期。

比如在tools/tests/selftests/mm/split_huge_page_test.c中is_backed_by_folio()就用到了这个文件提供的信息来判断进程地址后面对应的内存是不是一个大页。
