/proc/pid/pagemap文件保存了某个进程每个虚拟地址对应的内存状态。

# 代码

这个文件创建在fs/proc/base.c中，存放在tid_base_stuff[]数组中。也就是每个进程都该有的proc文件里.

对应到的operation是proc_pagemap_operations。当我们读取则合格文件时，调用的是其中的pagemap_read()函数。

# 格式

```
63      55 54   ...   0
┌────────┬────────────┐
│ 标志位  │   PFN/SWAP │
└────────┴────────────┘
```

文件中每64bit，存储了对应虚拟地址映射的物理页的状态。

比如：

   * pfn
   * 是否存在
   * 是否脏

具体可以参考pagemap_read()函数中的注释。

# 使用

比如在tools/testing/selftests/mm/split_huge_page_test.c中，就通过这个文件获取进程中虚拟地址的物理地址。
