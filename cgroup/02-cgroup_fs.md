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

回过头来我仔细一想才发现
