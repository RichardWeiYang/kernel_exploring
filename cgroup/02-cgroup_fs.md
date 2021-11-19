在上一节使用cgroup过程中，我们都是通过对文件的操作来达到目标。为什么对文件的操作就能创建cgroup，把进程加入到cgroup中呢？这就要说到cgroup文件系统了。

之所以采用文件系统的形式，而不是用系统调用的形式来支持cgroup功能，我想可能有两个原因：

  * 方便使用，不需要额外的库
  * 目录结构和进程树结构匹配

那我们就来看看cgroup的文件系统吧。

# 文件系统注册

首先在初始化时，注册了对应的文件系统：

  * cgroup_fs_type
  * cgroup2_fs_type

```
cgroup_init()
  register_filesystem(&cgroup_fs_type)
  register_filesystem(&cgroup2_fs_type)
```

从这里就看出，当前内核支持了两个版本的cgroup。

# 文件系统挂载

接下来的事情，应该发生在mount。但是本人暂时对mount的内容不熟，只能摘出我们当前能看得到的代码路径展示。

```
fsconfig()
  vfs_fsconfig_locked(fc)
    finish_clean_context(fc)
      fc->fs_type->init_fs_context(fc)
    vfs_get_tree(fc)
      fc->ops->get_tree(fc)
```

如果没有理解错，上述的路径将在每次mount的时候会执行一遍。这些操作主要做了这些事情：

  * 设置对应的fs->ops到cgroup_fs_context_ops/cgropu1_fs_context_ops
  * 创建了对应的cgroup_fs_context

执行的效果用下图展示。

![cgroup_fs_context](/cgroup/cgroup_fs_context.png)

回过头来我仔细一想才发现，这些操作的目的是**在get_tree()时绑定文件系统挂载点和cgroup层次结构**。也就是上图中的 fs_context->root 和 cgroup_fs_context->root。

## 谁挂载了cgroup

在上一节使用示例中我们直接在目录/sys/fs/cgroup上操作，也可以用mount查看到cgroup文件系统的挂载情况。

```
richard@richard:~$ mount | grep cgroup
tmpfs on /sys/fs/cgroup type tmpfs (ro,nosuid,nodev,noexec,mode=755)
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,name=systemd)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_cls,net_prio)
cgroup on /sys/fs/cgroup/rdma type cgroup (rw,nosuid,nodev,noexec,relatime,rdma)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpu,cpuacct)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
```

那究竟是谁挂载了这个目录呢？网上有很多文章说是systemd挂载的。

大致扫了一眼代码和注释，估计是下面的代码干的活。

```
static const MountPoint mount_table[] = {
    ...
    { "tmpfs",       "/sys/fs/cgroup",            "tmpfs",      "mode=755" TMPFS_LIMITS_SYS_FS_CGROUP,     MS_NOSUID|MS_NOEXEC|MS_NODEV|MS_STRICTATIME,
      cg_is_legacy_wanted, MNT_FATAL|MNT_IN_CONTAINER },
    ...
}

main()
  mount_setup_early()
    mount_points_setup()
      mount_one(mount_table + i, )
  initialize_runtime()
    mount_cgroup_controllers()
```

先挂载了/sys/fs/cgroup, 再挂载内核支持的cgroup。具体内核支持哪些cgroup从下面的文件获取。

```
# cat /proc/cgroups
#subsys_name	hierarchy	num_cgroups	enabled
cpuset	9	1	1
cpu	2	67	1
cpuacct	2	67	1
blkio	5	67	1
memory	10	140	1
devices	4	67	1
freezer	11	1	1
net_cls	8	1	1
perf_event	7	1	1
net_prio	8	1	1
hugetlb	3	1	1
pids	6	68	1
```

据说cgroupv2的挂载在另外一个地方，但是心急的我就先跳过这段吧。

# 相关文件创建

现在我们有了文件系统，也找到了是谁在哪里挂载了文件系统，接下来的问题就是那些文件从哪里来。

在上一节的例子中，我们通过写文件/sys/fs/cgroup/cpuset/test/cpuset.cpus来控制进程运行的cpu。那这个文件是哪里创建的呢？如果知道了这个文件后面的函数，我们不就知道linux是如何实现对进程调度的控制了吗。

这件事呢，说简单也简单，说复杂呢也有点复杂。因为相关文件的创建嵌套在整个cgroup系统的构造过程中。有些相互关联的概念会影响文件创建的过程。为了避免本节的内容太过发散，在这里我们主要来看创建文件过程中最有桥梁作用的函数**css_populate_dir**。

这个函数本身不长，为了较好理解，我把这个函数的核心部分贴上来。

```
static int css_populate_dir(struct cgroup_subsys_state *css)
{
	struct cgroup *cgrp = css->cgroup;
	struct cftype *cfts, *failed_cfts;

	if (!css->ss) {
		if (cgroup_on_dfl(cgrp))
			cfts = cgroup_base_files;
		else
			cfts = cgroup1_base_files;

		ret = cgroup_addrm_files(&cgrp->self, cgrp, cfts, true);
		if (ret < 0)
			return ret;
	} else {
		list_for_each_entry(cfts, &css->ss->cfts, node) {
			ret = cgroup_addrm_files(css, cgrp, cfts, true);
			if (ret < 0) {
				failed_cfts = cfts;
				goto err;
			}
		}
	}

	return 0;
```

上面的逻辑不难理解。一共有两种情况，但是最后都调用了cgroup_addrm_files()来创建对应的文件。而cgroup_addrm_files()展开为

```
cgroup_addrm_files(css, cgrp, cfts)
  cgroup_add_file(css, cgrp, cft)
    kn = __kernfs_create_file(cgrp->kn, cgroup_file_name(cgrp, cft, name),
      ..., cft->kf_ops, cft, ...)
```

这么看，这个函数的功能是

  * 在对应的cgroup目录下
  * 创建了一个名字为cgroup_file_name(cgrp, cft, name)的文件
  * 文件操作对应的回调函数是cft->kf_ops

知道了这些后，我们的问题就变成：

  * 这些cft是哪里来的？

再回到函数 css_populate_dir()， cft的来源根据css->ss的值分成两种情况。

  * cgroup基础文件
  * cgroup特定文件

## cgroup基础文件

cgroup基础文件就是每个cgroup目录下都会有的文件，如：

```
root@master:/sys/fs/cgroup# ls cpu/cgroup.*
cpu/cgroup.clone_children  cpu/cgroup.procs  cpu/cgroup.sane_behavior
root@master:/sys/fs/cgroup# ls cpuset/cgroup.*
cpuset/cgroup.clone_children  cpuset/cgroup.procs  cpuset/cgroup.sane_behavior
```

在cpu/cpuset目录下，我们都能看到这几个文件。而这些就是不论使用的是哪个cgroup都会有的文件。这些文件对应的cft就是cgroup1_base_files/cgroup_base_files。

打开cgroup1_base_files这个结构体一看，果然那几个文件名都在。（下面结构体略有删减）

```
/* cgroup core interface files for the legacy hierarchies */
struct cftype cgroup1_base_files[] = {
	{
		.name = "cgroup.procs",
		.seq_start = cgroup_pidlist_start,
		.seq_next = cgroup_pidlist_next,
		.seq_stop = cgroup_pidlist_stop,
		.seq_show = cgroup_pidlist_show,
		.private = CGROUP_FILE_PROCS,
		.write = cgroup1_procs_write,
	},
	{
		.name = "cgroup.clone_children",
		.read_u64 = cgroup_clone_children_read,
		.write_u64 = cgroup_clone_children_write,
	},
	{
		.name = "cgroup.sane_behavior",
		.flags = CFTYPE_ONLY_ON_ROOT,
		.seq_show = cgroup_sane_behavior_show,
	},
	{
		.name = "tasks",
		.seq_start = cgroup_pidlist_start,
	},
	{
		.name = "notify_on_release",
		.read_u64 = cgroup_read_notify_on_release,
		.write_u64 = cgroup_write_notify_on_release,
	},
	{
		.name = "release_agent",
		.flags = CFTYPE_ONLY_ON_ROOT,
	},
	{ }	/* terminate */
};
```

这样，我们想要了解echo $pid > cgroup.procs都发生了什么，是不是就知道去哪里找了呢？

## cgroup特定文件

顾名思义，特定文件就是某个cgroup特有的文件。 如：

```
root@master:/sys/fs/cgroup/cpuset# ll
total 0
dr-xr-xr-x  2 root root   0 Sep 21 01:17 ./
drwxr-xr-x 13 root root 340 Jul  3 13:40 ../
-rw-r--r--  1 root root   0 Nov 16 00:51 cgroup.clone_children
-rw-r--r--  1 root root   0 Nov 16 00:51 cgroup.procs
-r--r--r--  1 root root   0 Nov 16 00:51 cgroup.sane_behavior
-rw-r--r--  1 root root   0 Nov 16 00:51 cpuset.cpu_exclusive
-rw-r--r--  1 root root   0 Nov 16 00:51 cpuset.cpus
-r--r--r--  1 root root   0 Nov 16 00:51 cpuset.effective_cpus
-r--r--r--  1 root root   0 Nov 16 00:51 cpuset.effective_mems
-rw-r--r--  1 root root   0 Nov 16 00:51 cpuset.mem_exclusive
-rw-r--r--  1 root root   0 Nov 16 00:51 cpuset.mem_hardwall
-rw-r--r--  1 root root   0 Nov 16 00:51 cpuset.memory_migrate
-r--r--r--  1 root root   0 Nov 16 00:51 cpuset.memory_pressure
-rw-r--r--  1 root root   0 Nov 16 00:51 cpuset.memory_pressure_enabled
-rw-r--r--  1 root root   0 Nov 16 00:51 cpuset.memory_spread_page
-rw-r--r--  1 root root   0 Nov 16 00:51 cpuset.memory_spread_slab
-rw-r--r--  1 root root   0 Nov 16 00:51 cpuset.mems
-rw-r--r--  1 root root   0 Nov 16 00:51 cpuset.sched_load_balance
-rw-r--r--  1 root root   0 Nov 16 00:51 cpuset.sched_relax_domain_level
-rw-r--r--  1 root root   0 Nov 16 00:51 notify_on_release
-rw-r--r--  1 root root   0 Nov 16 00:51 release_agent
-rw-r--r--  1 root root   0 Nov 16 00:51 tasks
```

在cpuset目录下，还有很多以cpuset开头的文件。那他们都是从哪里来的呢？

从函数css_populate_dir()中，可以看到这里的cfts从css->ss->cfts上获得。而这个链表的初始化在下面的函数调用过程中：

```
cgroup_init()
  cgroup_add_dfl_cftypes(ss, ss->dfl_cftypes)
  cgroup_add_legacy_cftypes(ss, ss->legacy_cftypes)
    cgroup_add_cftypes()
      list_add_tail(&cfts->node, &ss->cfts);
```

也就是把ss->legacy_cftypes/dfl_cftypes挂载到了ss->cfts链表上。其表现形式用图表示为：

```
cgroup_subsys
+-------------------------------+<----------------------------------------+
|name                           |                                         |
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
|                               |
+-------------------------------+
```

看了一圈后，我们再回过头来看解答这个问题： 在cpuset目录下，以cpuset开头的文件是哪里来的？

对了，那就看cpuset_cgrp_subsys->dfl_ctypes/legacy_cftypes。

```
struct cgroup_subsys cpuset_cgrp_subsys = {
	.legacy_cftypes	= legacy_files,
	.dfl_cftypes	= dfl_files,
};
```

但是细心的朋友可能发现，legacy_files和dfl_files里面有重名的文件。那究竟是用的哪个呢？

那就要看cgroup_add_legacy_cftypes()/cgroup_add_dfl_cftypes()函数中给cft添加的flag，以及cgroup_addrm_files()中对flag判断的共同作用了～

## 文件与cgroup之间的关联

刚才我们看了cgroup_addrm_files()函数创建了cgroup相关文件，那这些文件是如何与cgroup产生关联的呢？

且看下图

![cgroup_files](/cgroup/cgroup_files.png)

从上面的图，我们可以得到：

  * cgroup的文件系统结构中，每个目录/文件用kernfs_node结构表示
  * 每个cgroup由一个目录和多个文件组成，从kernfs_node->priv指向对应的cgroup或cftype
  * 其中一个文件对应到一个cftype，文件的名字和对应的ops由对应的cftype决定
  * ops只有两种情况cgroup_kf_ops和cgroup_kf_single_ops，在cgroup_init_cftypes()中设置

# 如何通过文件配置cgroup

终于我们走到了正题。本节我们说了这么多就是为了研究在上一节中配置cgroup的方法究竟是如何生效的。在了解了cgroup的文件系统之后，我们终于可以来解答这个问题了。

上一节的例子中，配置cgroup分为三步：

  * 通过新建目录新建一个cgroup
  * 在cgroup.procs写入进程号
  * 根据不同的cgroup再做详细配置

其中第三步因cgroup的不同而不同，前两步对所有的都适用。在这里我们只看前两步，最后一步到时机成熟时再来看。

## 通过新建目录来新建cgroup

新建目录是通过mkdir实现的，这个好像前面没有提及？是的，在图cgroup_files中，我们隐藏了一个没有涉及的细节。kf_root中有个成员syscall_ops，分别可以被赋值为cgroup_kf_syscall_ops 和 cgroup1_kf_syscall_ops。

当我们打开这两个结构体一看：

```
+-----------------------+
|mkdir                  |  = cgroup_mkdir
|rmdir                  |  = cgroup_rmdir
|show_path              |  = cgroup_show_path
|show_options           |  = cgroup_show_options
+-----------------------+
```

我想你也猜到这些函数是干什么的了。

再来看一眼cgroup_mkdir吧

```
cgroup_mkdir()
  cgrp = cgroup_create(parent, name, mode)
  ret = css_populate_dir(&cgrp->self)
  ret = cgroup_apply_control_enable(cgrp)
  kernfs_activate(cgrp->kn)
```

我想，聪明如你，此时也不需要我多说什么了。

## cgroup.procs来添加进程

创建了cgroup后，第二步就是把进程移动到这个cgroup中。在上一节的例子中，我们是通过将进程号写入cgroup.procs文件来实现的。那就来看看这究竟是如何做到的吧。

在图cgroup_files中我们看到一个文件对应的一个kernfs_node结构体，

```
                                    cgroup1_base_files["cgroup.procs"]
                                +-->+-------------+
  kernfs_node                   |   |             |
  +------------------------+    |   |write        | = cgropu1_procs_write
  |parent                  |    |   |             |
  |   (struct kernfs_root*)|    |   |             |
  |priv                    |----+   +-------------+
  |   (void *)             |
  |                        |        cgroup_kf[_single]_ops
  |attr.ops                |------->+-------------+    
  |   (struct kernfs_ops *)|        |open         | = cgroup_file_open
  +------------------------+        |write        | = cgroup_file_write
                                    |poll         |
                                    |             |
                                    +-------------+
```

### kernfs_file_ops

因为cgroup的文件系统建立在kernfs基础上，所以对文件的读写操作实际上先要经过kernfs这一道。也就是要经过kernfs_file_ops中定义的操作。

```
const struct file_operations kernfs_file_fops = {
	.mmap		= kernfs_fop_mmap,
  .write_iter	= kernfs_fop_write_iter,
	.open		= kernfs_fop_open,
};
```

这些函数中又一个重要的helper， kernfs_ops：

```
static const struct kernfs_ops *kernfs_ops(struct kernfs_node *kn)
{
	if (kn->flags & KERNFS_LOCKDEP)
		lockdep_assert_held(kn);
	return kn->attr.ops;
}
```

看到这个kn->attr.ops了么？对cgroup的文件来说， ops就是cgroup_kf[_single]_ops。所以对cgroup文件的open/write，实际上就走到了cgroup_file_open/cgroup_file_write。
