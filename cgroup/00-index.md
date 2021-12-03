cgroup是计算机资源管理常用的管理设施，尤其是在云计算环境使用广泛。这里我们从cgroup的使用和架构上进行一次学习。

# 基本使用

对大多数人来说，接触cgroup还是使用cgroup来控制应用程序的资源。所以这一节，我们先来看看

[使用cgroup控制进程cpu和内存][1]

# cgroup文件系统

在上一节基本使用中，我们通过创建目录，向指定文件写入内容来使用cgroup。我想你也一定好奇，为什么这么操作就能达到这样的效果？

这一切都要归功于[cgroup文件系统][2]

# cgroup树形结构

从用户使用的角度看过了cgroup后，终于是时候揭开cgroup真实的面纱了。

在开始之前，我们先来好好看看cgroup的定义。来个借花献佛，我搬运一下[内核文档][3]

```
Control Groups provide a mechanism for aggregating/partitioning sets of
tasks, and all their future children, into hierarchical groups with
specialized behaviour.

Definitions:

A *cgroup* associates a set of tasks with a set of parameters for one
or more subsystems.

A *subsystem* is a module that makes use of the task grouping
facilities provided by cgroups to treat groups of tasks in
particular ways. A subsystem is typically a "resource controller" that
schedules a resource or applies per-cgroup limits, but it may be
anything that wants to act on a group of processes, e.g. a
virtualization subsystem.

A *hierarchy* is a set of cgroups arranged in a tree, such that
every task in the system is in exactly one of the cgroups in the
hierarchy, and a set of subsystems; each subsystem has system-specific
state attached to each cgroup in the hierarchy.  Each hierarchy has
an instance of the cgroup virtual filesystem associated with it.
```

从上面的描述中可以看到cgroup有两个重要的概念：

  * 层次结构
  * 子系统

层次结构是对系统中进程的切分，而子系统则赋予了cgroup在不同层面对进程的切变控制的方法。

每个子系统对系统的控制有着不同的方式，所以在本章我们主要讲述[cgroup的树形结构][4]。

虽然会涉及到一些子系统的内容，但其其中具体的精妙细节将留给各个子系统描述。

# cgroup和进程的关联

接下来我们需要了解的是cgroup和进程的关系。

从上面的定义中，我们看草cgroup联结了计算机系统中的两部分内容：

  * 进程
  * 系统资源

在上一节中，我们看到了cgroup的subsystem将系统的资源按照层次结构做了切分。（当然具体怎么切的需要看对应的subsystem，cgroup本身只提供了一种层次化结构框架。）那接下来就要看看内核是如何把进程和cgroup联系起来的。

[cgroup和进程的关联][5]

[1]: /cgroup/01-control_cpu_mem_by_cgroup.md
[2]: /cgroup/02-cgroup_fs.md
[3]: https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v1/cgroups.html
[4]: /cgroup/03-hierarchy.md
[5]: /cgroup/04-cgroup_and_process.md
