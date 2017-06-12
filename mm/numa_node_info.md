[NUMA(Non-uniform memory access)][1]是一个高级玩意，咱先来看看维基百科上的定义：

is a computer memory design used in multiprocessing, where the memory access time depends on the memory location relative to the processor. Under NUMA, a processor can access its own local memory faster than non-local memory (memory local to another processor or memory shared between processors). The benefits of NUMA are limited to particular workloads, notably on servers where the data are often associated strongly with certain tasks or users.

总的一句话来讲，就是分配内存时需要区别对待。不同区域的内存我们叫做不同的节点，节点之间我们又定义了个“距离”的概念用来衡量内存访问时间的差异。

好了，有了基本的概念，我们来看看linux内核中是如何获得物理内存的NUMA信息。说白了就是怎么知道一块内存对应到哪个NUMA节点。

在x86平台，这个工作分成两步

* 将numa信息保存到numa_meminfo
* 将numa_meminfo映射到memblock结构

本文着重关注第一次获取到numa信息的过程，对node和zone的数据结构暂时不在本文中体现。

# 全局概览

做事儿最好有点大局观，看代码学内核也是这个道理。那就先来看看总体结构。

```
setup_arch()
  initmem_init()
    x86_numa_init()
        numa_init()
            x86_acpi_numa_init()
            numa_cleanup_meminfo()
            numa_register_memblks()
```

接下来我们关注的就是最后这两个函数的作用。

# 将numa信息保存到numa_meminfo

在x86架构上，numa信息第一次获取是通过acpi或者是读取北桥上的信息。具体的函数是numa_init()。不管是哪种方式，numa相关的信息都最后保存在了numa_meminfo这个数据结构中。

这个数据结构和memblock长得很像，展开看就是一个数组，每个元素记录了一段内存的起止地址和node信息。

```
    numa_meminfo
    +------------------------------+
    |nr_blks                       |
    |    (int)                     |
    +------------------------------+
    |blk[NR_NODE_MEMBLKS]          |
    |    (struct numa_memblk)      |
    |    +-------------------------+
    |    |start                    |
    |    |end                      |
    |    |   (u64)                 |
    |    |nid                      |
    |    |   (int)                 |
    +----+-------------------------+
```

在这个过程中使用的就是这个函数添加的numa_meminfo数据结构。

* numa_add_memblk()

这个函数非常简单，而且粗暴。

# 将numa_meminfo映射到memblock结构

内核获取了numa_meminfo之后并没有如我们想象一般直接拿来用了。虽然此时给每个numa节点生成了我们以后会看到的node_data数据结构，但此时并没有直接使能它。

memblock是内核初期内存分配器，所以当内核获取了numa信息首先是把相关的信息映射到了memblock结构中，使其具有numa的knowledge。这样在内核初期分配内存时，也可以分配到更近的内存了。

在这个过程中有两个比较重要的函数

* numa_cleanup_meminfo()
* numa_register_memblks()

前者主要用来过滤numa_meminfo结构，合并同一个node上的内存。

后者就是把numa信息映射到memblock了。除此之外，顺便还把之后需要的node_data给分配了，为后续的页分配器做好了准备。

# 观察memblock的变化

memblock的调试信息默认没有打开，所以要观察的话需要传入内核启动参数"memblock=debug"。

进入系统后，输入命令"dmesg | grep -A 9 MEMBLOCK"可以看到

```
[    0.000000] MEMBLOCK configuration:
[    0.000000]  memory size = 0x00000001f9c4e800 reserved size = 0x0000000014aab27c
[    0.000000]  memory.cnt  = 0x7
[    0.000000]  memory[0x0]	[0x0000000000001000-0x000000000009cfff], 0x000000000009c000 bytes flags: 0x0
[    0.000000]  memory[0x1]	[0x0000000000100000-0x00000000ba5b1fff], 0x00000000ba4b2000 bytes flags: 0x0
[    0.000000]  memory[0x2]	[0x00000000ba5b9000-0x00000000bad8dfff], 0x00000000007d5000 bytes flags: 0x0
[    0.000000]  memory[0x3]	[0x00000000bafb6000-0x00000000ca8a1fff], 0x000000000f8ec000 bytes flags: 0x0
[    0.000000]  memory[0x4]	[0x00000000ca93a000-0x00000000ca977fff], 0x000000000003e000 bytes flags: 0x0
[    0.000000]  memory[0x5]	[0x00000000cafff000-0x00000000caffffff], 0x0000000000001000 bytes flags: 0x0
[    0.000000]  memory[0x6]	[0x0000000100000000-0x000000022f5fffff], 0x000000012f600000 bytes flags: 0x0
--
[    0.000000] MEMBLOCK configuration:
[    0.000000]  memory size = 0x00000001f9c4e800 reserved size = 0x0000000014b29800
[    0.000000]  memory.cnt  = 0x7
[    0.000000]  memory[0x0]	[0x0000000000001000-0x000000000009cfff], 0x000000000009c000 bytes on node 0 flags: 0x0
[    0.000000]  memory[0x1]	[0x0000000000100000-0x00000000ba5b1fff], 0x00000000ba4b2000 bytes on node 0 flags: 0x0
[    0.000000]  memory[0x2]	[0x00000000ba5b9000-0x00000000bad8dfff], 0x00000000007d5000 bytes on node 0 flags: 0x0
[    0.000000]  memory[0x3]	[0x00000000bafb6000-0x00000000ca8a1fff], 0x000000000f8ec000 bytes on node 0 flags: 0x0
[    0.000000]  memory[0x4]	[0x00000000ca93a000-0x00000000ca977fff], 0x000000000003e000 bytes on node 0 flags: 0x0
[    0.000000]  memory[0x5]	[0x00000000cafff000-0x00000000caffffff], 0x0000000000001000 bytes on node 0 flags: 0x0
[    0.000000]  memory[0x6]	[0x0000000100000000-0x000000022f5fffff], 0x000000012f600000 bytes on node 0 flags: 0x0
```

注意看每个memory条目的最后，现在每个memblock上都多了一个输出"on node 0"。这就是物理内存的NUMA信息。当然我这里只有一个node，如果你的系统上有多个node那就可以看到不一样的信息了。

# 插播一条我的补丁

在code review过程中，发现numa_nodemask_from_meminfo()的作用其实可以被别的东西替代。所以我干脆提了个补丁把这个挪走了。

补丁的id是：[474aeffd88b87746a75583f356183d5c6caa4213][2]

不过可惜的是，这个补丁还有点小问题。新的改进也写好了，但是到现在还没有合入。社区嘛，你懂的。

[1]: https://en.wikipedia.org/wiki/Non-uniform_memory_access
[2]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=474aeffd88b87746a75583f356183d5c6caa4213
