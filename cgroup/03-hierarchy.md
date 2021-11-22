cgroup形成的最重要的结构就是层次结构，也就是树形结构。其实你想一下，这是一个非常自然的结果。因为cgroup管理的对象就是进程树，那么自然自己也展现成一颗树的样子。

想到了这一层，那cgroup的树形结构就比较好理解，且理论上没有什么好讲的。但是有意思的是，cgroup的树形结构并不是以cgroup本身链接起来的，而是由subsystem的状态链接起来的。据说是为了更快捷的访问需要控制的状态数据。所以在介绍cgroup的树形结构中，我们要讲到

  * cgroup
  * subsystem
  * subsystem_state

# subsystem的全局定义

在正式开始介绍cgroup前，我们先来看看几个全局变量的定义。因为cgroup所支持的subsystem不能像驱动那样随意添加，所以内核干脆把支持的subsystem定义成了全局变量。通过对这些全局变量的了解，我们也能知道当前内核究竟支持了多少subsystem。

内核中一共定义了三组全局变量：

  * cgroup_subsys_id
  * cgroup_subsys_name[]
  * cgroup_subsys[]

其实很简单，但是内核开发者用了一些很有意思的手法来定义，所以当你第一次找的时候不一定能找到。那就让我来为你慢慢揭开这个面纱。

首先这三个全局变量的定义形式都很类似：

```
#define SUBSYS(_x) _x ## _cgrp_id,
enum cgroup_subsys_id {
#include <linux/cgroup_subsys.h>
	CGROUP_SUBSYS_COUNT,
};

#define SUBSYS(_x) [_x ## _cgrp_id] = #_x,
static const char *cgroup_subsys_name[] = {
#include <linux/cgroup_subsys.h>
};

#define SUBSYS(_x) [_x ## _cgrp_id] = &_x ## _cgrp_subsys,
struct cgroup_subsys *cgroup_subsys[] = {
#include <linux/cgroup_subsys.h>
};
```

也就是都先定义了一个宏 SUBSYS，然后引用了头文件 linux/cgroup_subsys.h。其实头文件的内容很简单，就是一个个SUBSYS加上当前内核支持的subsystem的名字。

讲到这里可能还有点模糊，那我就干脆手工把这些定义展开，来看个明明白白。

## cgroup_subsys_id

```
enum cgroup_subsys_id {
	cpuset_cgrp_id,
	cpu_cgrp_id,
	cpuacct_cgrp_id,
	io_cgrp_id,
	memory_cgrp_id,
	devices_cgrp_id,
	freezer_cgrp_id,
	net_cls_cgrp_id,
	perf_event_cgrp_id,
	net_prio_cgrp_id,
	hugetlb_cgrp_id,
	pids_cgrp_id,
	rdma_cgrp_id,
	debug_cgrp_id,
	CGROUP_SUBSYS_COUNT,
};
```

## cgroup_subsys_name[]

```
static const char *cgroup_subsys_name[] = {
	[cpuset_cgrp_id]       =      "cpuset",
	[cpu_cgrp_id]          =      "cpu",
	[cpuacct_cgrp_id]      =      "cpuacct",
	[io_cgrp_id]           =      "io",
	[memory_cgrp_id]       =      "memory",
	[devices_cgrp_id]      =      "devices",
	[freezer_cgrp_id]      =      "freezer",
	[net_cls_cgrp_id]      =      "net_cls",
	[perf_event_cgrp_id]   =      "perf_event",
	[net_prio_cgrp_id]     =      "net_prio",
	[hugetlb_cgrp_id]      =      "hugetlb",
	[pids_cgrp_id]         =      "pids",
	[rdma_cgrp_id]         =      "rdma",
	[debug_cgrp_id]        =      "debug",
};
```

## cgroup_subsys[]

```
struct cgroup_subsys *cgroup_subsys[] = {

	[cpuset_cgrp_id]       =      cpuset_cgrp_subsys,
	[cpu_cgrp_id]          =      cpu_cgrp_subsys,
	[cpuacct_cgrp_id]      =      cpuacct_cgrp_subsys,
	[io_cgrp_id]           =      io_cgrp_subsys,
	[memory_cgrp_id]       =      memory_cgrp_subsys,
	[devices_cgrp_id]      =      devices_cgrp_subsys,
	[freezer_cgrp_id]      =      freezer_cgrp_subsys,
	[net_cls_cgrp_id]      =      net_cls_cgrp_subsys,
	[perf_event_cgrp_id]   =      perf_event_cgrp_subsys,
	[net_prio_cgrp_id]     =      net_prio_cgrp_subsys,
	[hugetlb_cgrp_id]      =      hugetlb_cgrp_subsys,
	[pids_cgrp_id]         =      pids_cgrp_subsys,
	[rdma_cgrp_id]         =      rdma_cgrp_subsys,
	[debug_cgrp_id]        =      debug_cgrp_subsys,

};
```

ok,这样当内核中出现这些变量的时候，你就知道都对应到是谁，要去哪里找他们了吧。
