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
