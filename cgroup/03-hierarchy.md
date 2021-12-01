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

# cgroup初始化概览

终于要揭开cgroup的面纱了，我们先来看cgroup是在什么地方初始化的，以及初始化大致都做了些什么工作。

```
start_kernel()
    cgroup_init_early()
        init_cgroup_root(&ctx)
            init_cgroup_housekeeping(cgrp)
        for_each_subsys(ss, i) {
            if (ss->early_init)
                cgroup_init_subsys(ss)
        }
    cgroup_init()
        cgroup_init_cftypes(NULL, cgroup_base_files)
        cgroup_init_cftypes(NULL, cgroup1_base_files)
        cgroup_setup_root(&cgrp_dfl_root, 0)
        for_each_subsys(ss, ssid) {
            if (!ss->early_init)
                cgroup_init_subsys(ss)
            cgroup_add_dfl_cftypes(ss, ss->dfl_cftypes)
            cgroup_add_legacy_cftypes(ss, ss->legacy_cftypes)
            if (ss->bind)
                ss->bind(init_css_set.subsys[ssid])
            css_populate_dir(init_css_set.subsys[ssid])
        }
        sysfs_create_mount_point(fs_kobj, "cgroup")
        register_filesystem(&cgroup_fs_type)
        register_filesystem(&cgroup2_fs_type)
```

从上面摘出的代码来看，初始化cgroup主要做了这么几件事：

  * 注册cgroup文件系统，及初始化对应的cft
  * 初始化cgroup_root
  * 初始化每个配置了的subsystem

因为cgroup文件系统相关内容我们已经在上一节中详细阐述，所以接下来我们就看看cgroup_root和subsystem的初始化。

# cgroup_root的初始化

和cgroup_root初始化相关的有两个地方

  * init_cgroup_root()
  * cgroup_setup_root()

在上一节中我们看到过cgroup_root结构体，但我们着重看了和cgroup文件系统相关的部分。现在我们来看一下这个结构体的全貌。

```
cgroup_roots(struct list_head)
  |
  |       cgroup_root(cgrp_dfl_root)
  |       +-------------------------------+<----+
  +------>|root_list                      |     |
          |    (struct list_head)         |     |
          |name[]                         |     |
          |    (char)                     |     |
          |hierarchy_id                   |     |
          |    (int)                      |     |
          |                               |     |
          |cgrp                           |     |
          |    (struct cgroup)            |     |
          |    +--------------------------+<----|--------------------------+
          |    |root                   ---|-----+                          |
          |    |   (struct cgroup_root*)  |                                |
          |    |                          |                                |
          |    |kn                        |  = kf_root->kn                 |
          |    |    (struct kernfs_node *)|-----+                          |
          |    +--------------------------+     |                          |
          |kf_root                        |     |                          |
          |    (struct kernfs_root*)      |     |                          |
          |    +--------------------------+     |                          |
          |    |kn                        |     |                          |
          |    |  (struct kernfs_node*)   |     |                          |
          |    |  +-----------------------+<----+                          |
          |    |  |priv                   |--------------------------------+
          |    |  |  (void *)             |  = cgrp_dfl_root.cgrp
          |    +--+-----------------------+
          |    |syscall_ops               |  = cgroup_kf_syscall_ops | cgroup1_kf_syscall_ops
          |    |  (struct kernfs_syscall*)|
          |    |  +-----------------------+
          |    |  |mkdir                  |  = cgroup_mkdir
          |    |  |rmdir                  |  = cgroup_rmdir
          |    |  |show_path              |  = cgroup_show_path
          |    |  |show_options           |  = cgroup_show_options
          |    |  +-----------------------+
          |    +--------------------------+
          |                               |
          |subsys_mask                    |  bitmask of subsys attached
          |    (unsigned int)             |  set in cgroup_init() for cgrp_dfl_root
          |                               |      in rebind_subsystems() for others
          |nr_cgrps                       |
          |    (atomic_t)                 |
          +-------------------------------+
```

从结构体的定义上可以看出，cgroup_root主要包含了：

  * 一个根cgroup
  * 一个kernfs文件系统的根结构

而cgroup_root的初始化主要是准备好这两个结构体，也就是设置好了cgroup文件系统目录结构并关联到了某个cgroup树形结构的根上。

# subsystem的初始化

先来看一下subsystem初始化都做了哪些事儿。

```
cgroup_init_subsys(ss, early)
    ss->root = &cgrp_dfl_root
    css = ss->css_alloc()
    init_and_link_css(css, ss, &cgrp_dfl_root.cgrp)
        css->cgroup = cgrp
        css->ss = ss
    init_css_set.subsys[ss->id] = css
    online_css(css)
        ss->css_online(css)
        css->cgroup->subsys[ss->id] = css
```

嗯，看上去简直就是天书，完全不知道是在干啥。ok，让我来拆解一下。这段代码中涉及到四个数据结构：

  * cgroup_root
  * cgroup
  * cgroup_subsys
  * cgroup_subsys_state

而这个函数要做的事儿，就是把这四个结构体关联起来。那就让我们慢慢讲来：

## cgroup_subsys的模样

```
cgroup_subsys
+-------------------------------+<----------------------------------------+
|name                           |                                         |
|legacy_name                    |                                         |
|    (char *)                   |                                         |
|id                             |                                         |
|    (int)                      |                                         |
|                               |                                         |
|root                           |  = &cgrp_dfl_root                       |
|    (struct cgroup_root*)      |    init in cgroup_init_subsys()         |
|                               |    rebind in rebind_subsystems()        |
|                               |                                         |
|dfl_cftypes                    |  be populated by css_populate_dir()     |
|legacy_cftypes                 |                                         |
|    (struct cftype*)           |                                         |
|                               |    dfl_cftypes       legacy_cftypes     |
|                               |    +------------+    +------------+     |
|cfts                        ---|--->|node     ---|--->|node        |     |
|    (struct list_head)         |    |ss          |    |ss       ---|-----+
|                               |    |kf_ops      |    |kf_ops      |
|                               |    +------------+    +------------+    
+-------------------------------+
```

从这个图上看，subsys只和cgroup_root之前存在联系，而且只和一个cgroup_root有关。这也解释了系统中对某一个subsys的系统划分只能有一种。

另外我们也看到了，某个subsys自带的配置文件是关联在subsys上的。由函数cgroup_add_legacy_cftypes/cgroup_add_dfl_cftypes在cgroup_init()时添加到了ss->cfts链表。

## cgroup和cgroup_subsys_state

```
cgroup
+--------------------------------------+<--------------------------+
|subsys[CGROUP_SUBSYS_COUNT]           |                           |
|    (struct cgroup_subsys_state*)     |                           |
|    [0]  +----------------------------+                           |
|         |cgroup                      |---------------------------+
|         |   (struct cgroup*)         |                           |
|         |ss                          |  = cgroup_subsys[0]       |
|         |   (struct cgroup_subsys*)  |                           |
|    [1]  +----------------------------+                           |
|         |cgroup                      |---------------------------+
|         |   (struct cgroup*)         |                           |
|         |ss                          |  = cgroup_subsys[1]       |
|         |   (struct cgroup_subsys*)  |                           |
|    [2]  +----------------------------+                           |
|         |                            |                           |
|         .                            .                           |
|         .                            .                           |
|         |                            |                           |
|    [15] +----------------------------+                           |
|         |cgroup                      |---------------------------+
|         |   (struct cgroup*)         |
|         |ss                          |  = cgroup_subsys[15]
|         |   (struct cgroup_subsys*)  |
|         |                            |
+---------+----------------------------+
```

cgroup上有一个长度为CGROUP_SUBSYS_COUNT的数组， subsys[]。其中每一个成员都指向了一个cgroup_subsys_state结构。

从这个结构上可以看出：

  * 每个cgroup可以关联所有的subsys
  * 如果对应的成员为NULL，表示该cgroup没有和对应的subsys绑定
  * 也可以看到每个subsys_state都有指向自己是属于哪个subsys的指针

## cgroup_subsys的层次结构

```
                     cgroup
                     +--------------------------------------+
                     |subsys[id]                        ^   |
                     |    (struct cgroup_subsys_state*) |   |
                     |    +---------------------------------+
                     |    |cgroup                    ---+   |
                     |    |    (struct cgroup*)             |
                     |    |ss                               |
                     |    |    (struct cgroup_subsys*)      |  = non-NULL
                     |    |                                 |
                     |    |parent                           |
                     |    |    (struct cgroup_subsys_state*)|
                     |    |                                 |
                     |    |children                ||       |
                     |    |    (struct list_head)  ||       |
                     +----+---------------------------------+ <--------------------------------------+
                                              ^    ||                                                |
       	                                      |    ||                                                |
                                              |    ||                                                |
                                              |    ||                                                |
                                              |    ||                                                |
    cgroup                                    |    ||      cgroup                                    |
    +--------------------------------------+  |    ||      +--------------------------------------+  |
    |subsys[id]                        ^   |  |    ||      |subsys[id]                        ^   |  |
    |    (struct cgroup_subsys_state*) |   |  |    ||      |    (struct cgroup_subsys_state*) |   |  |
    |    +---------------------------------+  |    ||      |    +---------------------------------+  |
    |    |cgroup                    ---+   |  |    ||      |    |cgroup                    ---+   |  |
    |    |    (struct cgroup*)             |  |    ||      |    |    (struct cgroup*)             |  |
    |    |                                 |  |    ||      |    |                                 |  |
    |    |parent                        ---|--+    ||      |    |parent                        ---|--+
    |    |    (struct cgroup_subsys_state*)|       ||      |    |    (struct cgroup_subsys_state*)|
    |    |                                 |       ||      |    |                                 |
    |    |sibling                      ====|=======++======|====|sibling                          |
    |    |    (struct list_head)           |               |    |    (struct list_head)           |
    +----+---------------------------------+               +----+---------------------------------+
```

实际上这个工作主要是在css_create()函数中做到的，但是因为这个确实比较重要，我就把他放在了这里。

怎么样，现在是不是有点感觉了？我们已经在subsys_state的角度形成了一个层次结构。这是不是已经很像我们用mkdir创建cgroup时形成的层次结构？

## 隐藏的cgroup_subsys_state

最后要放一张神奇的图：

```
                     cgroup
                     +--------------------------------------+
                     |self                              ^   |
                     |    (struct cgroup_subsys_state)  |   |
                     |    +---------------------------------+
                     |    |cgroup                    ---+   |
                     |    |    (struct cgroup*)             |
                     |    |ss                               |
                     |    |    (struct cgroup_subsys*)      |  = NULL
                     |    |parent                           |
                     |    |    (struct cgroup_subsys_state*)|
                     |    |                                 |
                     |    |children                ||       |
                     |    |    (struct list_head)  ||       |
                     +----+---------------------------------+ <--------------------------------------+
                                              ^    ||                                                |
       	                                      |    ||                                                |
                                              |    ||                                                |
                                              |    ||                                                |
                                              |    ||                                                |
    cgroup                                    |    ||      cgroup                                    |
    +--------------------------------------+  |    ||      +--------------------------------------+  |
    |self                              ^   |  |    ||      |self                              ^   |  |
    |    (struct cgroup_subsys_state)  |   |  |    ||      |    (struct cgroup_subsys_state)  |   |  |
    |    +---------------------------------+  |    ||      |    +---------------------------------+  |
    |    |cgroup                    ---+   |  |    ||      |    |cgroup                    ---+   |  |
    |    |    (struct cgroup*)             |  |    ||      |    |    (struct cgroup*)             |  |
    |    |                                 |  |    ||      |    |                                 |  |
    |    |parent                        ---|--+    ||      |    |parent                        ---|--+
    |    |    (struct cgroup_subsys_state*)|       ||      |    |    (struct cgroup_subsys_state*)|
    |    |                                 |       ||      |    |                                 |
    |    |sibling                      ====|=======++======|====|sibling                          |
    |    |    (struct list_head)           |               |    |    (struct list_head)           |
    +----+---------------------------------+               +----+---------------------------------+
```

这张图是不是和上面那张很像？几乎一模一样？

是的，唯一的区别在于这次链接起来的是cgroup->self，而不是cgroup->subsys[]。

这个问题我也没有想明白，为啥还要单独用一个隐藏的subsys_state来链接？为啥不直接用cgroup来链接？我能想到的解释是遗留问题。可能最开始没有想到要加这么多subsys。

好了，cgroup的层次结构就这么突然的呈现在你的面前，没有什么明显的征兆。就好像生活中某些事儿突如其来又戛然而止。

# 相关函数

对整个结构有了全貌后，我们对一些常用的函数做一些了解。这样可以加深对整个结构的了解，同时也可以验证我们之前的认识是否正确。

我们将看一些比较常用的函数，如：

  * 遍历层次结构
  * 创建cgroup
  * 创建css

## 遍历层次结构

既然我们构建了一个cgroup的层次结构，那么必然需要遍历这个结构。内核中提供了三个帮助我们遍历的函数：

  * cgroup_for_each_live_child()
  * cgroup_for_each_live_descendant_pre()
  * cgroup_for_each_live_descendant_post()

这三个都是宏定义，且不长。我们就都打开看一下。

```
/* iterate over child cgrps, lock should be held throughout iteration */
#define cgroup_for_each_live_child(child, cgrp)				\
	list_for_each_entry((child), &(cgrp)->self.children, self.sibling) \
		if (({ lockdep_assert_held(&cgroup_mutex);		\
		       cgroup_is_dead(child); }))			\
			;						\
		else

/* walk live descendants in pre order */
#define cgroup_for_each_live_descendant_pre(dsct, d_css, cgrp)		\
	css_for_each_descendant_pre((d_css), &(cgrp)->self)		\
		if (({ lockdep_assert_held(&cgroup_mutex);		\
		       (dsct) = (d_css)->cgroup;			\
		       cgroup_is_dead(dsct); }))			\
			;						\
		else

/* walk live descendants in postorder */
#define cgroup_for_each_live_descendant_post(dsct, d_css, cgrp)		\
	css_for_each_descendant_post((d_css), &(cgrp)->self)		\
		if (({ lockdep_assert_held(&cgroup_mutex);		\
		       (dsct) = (d_css)->cgroup;			\
		       cgroup_is_dead(dsct); }))			\
			;						\
		else
```

可以看到，第一个遍历其实就是遍历了cgrp->self.children这个链表。也就这一个层次，遍历是比较简单的。

而后两者是遍历整个子树了，所以一眼看不清楚具体做了啥。但是都调用了一个非常像的函数css_for_each_descendant_[pre|post]。

让我们来看看其中一个的实现。

```
#define css_for_each_descendant_pre(pos, css)				\
	for ((pos) = css_next_descendant_pre(NULL, (css)); (pos);	\
	     (pos) = css_next_descendant_pre((pos), (css)))

struct cgroup_subsys_state *
css_next_descendant_pre(struct cgroup_subsys_state *pos,
			struct cgroup_subsys_state *root)
{
	struct cgroup_subsys_state *next;

	cgroup_assert_mutex_or_rcu_locked();

	/* if first iteration, visit @root */
	if (!pos)
		return root;

	/* visit the first child if exists */
	next = css_next_child(NULL, pos);
	if (next)
		return next;

	/* no child, visit my or the closest ancestor's next sibling */
	while (pos != root) {
		next = css_next_child(pos, pos->parent);
		if (next)
			return next;
		pos = pos->parent;
	}

	return NULL;
}
```

其中css_next_child()只是寻找parent下pos后面一个孩子，如果都找到了，则网上一层。这样配合起来就完成了子树的遍历。

## 创建cgroup

接下来一个重要的函数是创建cgroup的了。

```
cgroup_create(parent)
    cgrp = kzalloc()
    percpu_ref_init(&cgrp->self.refcnt, css_release, 0, GFP_KERNEL);
    cgroup_rstat_init(cgrp);
    kn = kernfs_create_dir(parent->kn, name, mode, cgrp)
    init_cgroup_housekeeping(cgrp)
    list_add_tail_rcu(&cgrp->self.sibling, &cgroup_parent(cgrp)->self.children);
    cgroup_propagate_control(cgrp)
```

这个函数里，创建了一个空的cgroup结构，然后把这个结构体收拾干净。备用。

那给谁用呢？还记得之前看过的cgroup_mkdir么？对了，就是给他用。

```
cgroup_mkdir()
  cgrp = cgroup_create(parent, name, mode)
  ret = css_populate_dir(&cgrp->self)
  ret = cgroup_apply_control_enable(cgrp)
  kernfs_activate(cgrp->kn)
```

扫了一眼，大部分我们已经心中有数了。无非是创建cft文件，并显示。其中只有一个函数我们还不清楚。cgroup_apply_control_enable。

```
static int cgroup_apply_control_enable(struct cgroup *cgrp)
{
	struct cgroup *dsct;
	struct cgroup_subsys_state *d_css;
	struct cgroup_subsys *ss;
	int ssid, ret;

	cgroup_for_each_live_descendant_pre(dsct, d_css, cgrp) {
		for_each_subsys(ss, ssid) {
			struct cgroup_subsys_state *css = cgroup_css(dsct, ss);

			if (!(cgroup_ss_mask(dsct) & (1 << ss->id)))
				continue;

			if (!css) {
				css = css_create(dsct, ss);
				if (IS_ERR(css))
					return PTR_ERR(css);
			}

			WARN_ON_ONCE(percpu_ref_is_dying(&css->refcnt));

			if (css_visible(css)) {
				ret = css_populate_dir(css);
				if (ret)
					return ret;
			}
		}
	}

	return 0;
}
```

遍历cgroup子树的函数cgroup_for_each_live_descendant_pre()我们刚看过，因为这个cgroup是刚创建的，所以这里就遍历了一个节点。而整个函数的目标还是创建对应的cft文件并显示。只不过这里还隐含了一个动作css_create()。是的，之前我们看到的cgroup_subsystem_state就是在这里创建的。

## 创建css
