cgroup是计算机资源管理常用的管理设施，尤其是在云计算环境使用广泛。这里我们从cgroup的使用和架构上进行一次学习。

# 基本使用

对大多数人来说，接触cgroup还是使用cgroup来控制应用程序的资源。所以这一节，我们先来看看

[使用cgroup控制进程cpu和内存][1]

# cgroup文件系统

在上一节基本使用中，我们通过创建目录，向指定文件写入内容来使用cgroup。我想你也一定好奇，为什么这么操作就能达到这样的效果？

这一切都要归功于[cgroup文件系统][2]

# cgroup树形结构

# cgropu和进程


[1]: /cgroup/01-control_cpu_mem_by_cgroup.md
[2]: /cgroup/02-cgroup_fs.md
