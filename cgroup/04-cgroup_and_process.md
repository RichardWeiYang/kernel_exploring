cgroup最终用来控制进程资源的使用，讲了这么半天一直都缺一个主角，那就是进程。不是要讲述进程本身，而是要理解进程如何和cgroup关联。

在[内核文档][3]有这么一句话

```
Each task in the system has a reference-counted pointer to a css_set.
```

那我们就从css_set开始探索。

# css_set的静态结构

```
css_set_table              css_set
+---------------+          +-------------------------------+
|               |     +--->|hlist                          |
|               |     |    |    (struct hlist_node)        |
+---------------+     |    |                               |
|css_set_hash() |-----+    |refcount                       |
|               |          |    (refcount_t)               |
+---------------+          |dom_cset                       |
|               |          |    (struct css_set*)          |
|               |          |dfl_cgrp                       |
+---------------+          |    (struct cgroup*)           |
                           |                               |
                           |subsys[CGROUP_SUBSYS_COUNT]    |
                           |  (struct cgroup_subsys_state*)|
                           |  +----------------------------+
                           |  |ss                          |
                           |  |    (struct cgroup_subsys*) |
                           |  +----------------------------+
                           +-------------------------------+
```

这么一看是不是很简单？错了，我只是把最简单的部分拿了出来。其中我们可以看到的是

  * css_set用了一张哈希表保存
  * 一个css_set中有一个完整的cgroup_subsys_state列表

有了这个概念后，我们再来看这个css_set究竟关联了谁。

# css_set的动态演化

css_set动态演化是一个不容易描述的过程，不仅仅是单个css_set的变化，更是整个css_set这个集合的变化。

你想，为啥会有一个哈希表来保存css_set呢？这其实是css_set和cgroup之间的映射关系。好了，那就让我们来看看把。

## cgrp_cset_link

在动起来之前，还要再插入一个小知识。那就是cgrp_cset_link结构体。内核中结构体就是多，不知道它，还真没法说清楚。

```
    cgrp_cset_link
    +-------------------------------+
    |cgrp                           |
    |    (struct cgroup*)           |
    |cset                           |
    |    (struct css_set*)          |
    |                               |
    |cset_link                      |
    |cgrp_link                      |
    |    (struct list_head)         |
    +-------------------------------+
```

结构体本身很简单，一头连着cgroup,一头连着css_set。把他们串起来后呢长这个样子。

```
    css_set
    +--------------------+------------+-----------------------------+
    |cgrp_links          |            |                             |
    +--------------------+            |                             |
       |                              |                             |
       |                              |                             |
       |      cgrp_cset_link          |     cgrp_cset_link          |
       |      +--------------------+  |     +--------------------+  |
       |      |cset            ----|--+     |cset             ---|--+
       +----->|cgrp_link           |------->|cgrp_link           |
              |cgrp  ---+          |        |cgrp  ---+          |
              +--------------------+        +--------------------+
                        |                             |
              cgroup    v                   cgroup    v
              +--------------------+        +--------------------+
              |                    |        |                    |
              +--------------------+        +--------------------+


    cgroup
    +--------------------+------------+-----------------------------+
    |cset_links          |            |                             |
    +--------------------+            |                             |
       |                              |                             |
       |                              |                             |
       |      cgrp_cset_link          |     cgrp_cset_link          |
       |      +--------------------+  |     +--------------------+  |
       |      |cgrp            ----|--+     |cgrp             ---|--+
       +----->|cset_link           |------->|cset_link           |
              |cset  ---+          |        |cset  ---+          |
              +--------------------+        +--------------------+
                        |                             |
              css_set   v                   css_set   v
              +--------------------+        +--------------------+
              |                    |        |                    |
              +--------------------+        +--------------------+
```

说白了呢就是实现了cgropu和css_set之间的互相访问。一句话，你中有我，我中有你。

## link_css_set()

实现上面这样连接效果靠的是函数link_css_set()。而这个函数只有在两个地方被调用

  * cgroup_setup_root
  * find_css_set

当我们仔细看这两种情况下的操作，你就会发现css_set是怎么样和cgroup之间发生联系的了。

理论上最好使用三维图来表示css_set和cgroup之间的关系，但因为小编制图技术有限，暂且用二维图像来说明。大家可以脑补一下。

下图是刚创建cgroup_root时的一个映射关系。

```
             cgroup_root1                 cgroup_root2
             +-----------+                +-----------+
             |           |                |           |
             +-----------+                +-----------+
                     ^                       ^
                     |                       |
                     |                       |
               +--------------+        +--------------+
               |cgrp  ---+    |        |cgrp  ---+    |
        +----->|cgrp_link     |------->|cgrp_link     |
        |      +--------------+        +--------------+
        |      cgrp_cset_link          cgrp_cset_link
        |
     +--------------------+
     |cgrp_links          |
     +--------------------+
     css_set1

```

下图显示了当有cgroup层次结构后的一个映射关系。

```
             cgroup_root1                 cgroup_root2
             +-----------+                +-----------+
             |           |                |           |
             +-----------+                +-----------+
                /    \                           ^
              /        \                         |
            /            \                       |
     +-----------+   +-----------+               |
     |           |   |           |               |
     +-----------+   +-----------+               |
            ^                                    |
            |                                    |
            +------------+                       |
                         |                       |
                         |                       |
                         |                       |
               +--------------+        +--------------+
               |cgrp  ---+    |        |cgrp  ---+    |
        +----->|cgrp_link     |------->|cgrp_link     |
        |      +--------------+        +--------------+
        |      cgrp_cset_link          cgrp_cset_link
        |
     +--------------------+
     |cgrp_links          |
     +--------------------+
     css_set1
```

当我第一次理解这种映射关系的时候，我有种震撼的感觉。假如你把cgroup的层次结构想象成一个平面，而css_set在其上的另一个明面。那么任意一个css_set就会有来自所有cgroup_root子树上某一个cgroup的连线。而我们换一个角度思考，一个css_set代表了一个进程的cgroup配置，而每一个css_set对每一个cgroup_root的关联就表示了这个进程在这个cgroup subsystem上的配置。或者说，一个css_set就是所有cgroup的一个快照。

# 进程在cgroup间的移动

上一小节动态演化中，cgroup和进程(css_set)之间的关系可以由函数find_css_set()引导出来。可以分为：

  * 进程创建
  * 进程移动

进程创建时默认加入到和傅进程同样的cgroup中，所以find_css_set()还是会沿用父进程的css_set()。
而进程创建则会复杂一些，在可能创建新css_set的同时，也隐藏着更多的技巧和细节。

## 从文件系统接口开始

在第一小节cgroup控制进程中我们就看到了如何增加一个cgroup，并将进程加入到这个cgroup中。这里就是进程在cgroup间移动的入口。

既然已经学过了cgroup文件系统，那就让我们从这里开始吧。

```
    cgroup.procs, __cgroup1_procs_write()
        cgrp = cgroup_kn_lock_live(of->kn, false)
        task = cgroup_procs_write_start(buf, threadgroup, )
            kstrtoint(strstrip(buf), 0, &pid)
            tsk = find_task_by_vpid(pid)
            get_task_struct(tsk)
        cgroup_attach_task(cgrp, task, threadgroup)
            cgroup_migrate_add_src(task_css_set(task), cgrp, &mgctx)
            cgroup_migrate_prepare_dst()
            cgroup_migrate(leader, threadgroup, )
                cgroup_migrate_add_task(task, mgctx)
                cgroup_migrate_execute(mgctx)
            cgroup_migrate_finish(&mgctx)
        cgroup_procs_write_finish(task, locked)
            put_task_struct(task)
            ss->post_attach()
```

上面这个函数__cgroup1_procs_write()就是当我们写入一个pid后，cgroup系统会做的事儿。主要来讲分成三块：

  * 从pid找到对应的进程
  * 将进程转移到目标cgroup
  * 一些清理工作

很明显，和cgroup相关的操作在第二步中。

## 一张图来解释

cgroup_attach_task()这个函数的工作我尝试用一张图来解释：

```
                                      cgroup_root1
                                      +-----------+
                                      |           |
                                      +-----------+
                                         /    \
                                       /        \
                                     /            \
                              +-----------+   +-----------+
                              |cgrp1      |   |cgrp2      |
                              +-----------+   +-----------+
                                     ^           ^
                                     |           |
     task                            +----+      |
     +--------+                           |      |
     |cgroups |--->src_cset               |      |          dst_cset            
     +--------+    +------------------+   |      |     +--->+------------------+
                   |mg_src_cgrp    ---|---+      |     |    |mg_src_cgrp       |
                   |mg_dst_cgrp    ---|----------+     |    |mg_dst_cgrp       |
                   |  (struct cgroup*)|                |    |  (struct cgroup*)|
                   |                  |                |    |                  |
                   |mg_dst_cset    ---|----------------+    |mg_dst_cset       |
                   | (struct css_set*)|                     | (struct css_set*)|
                   |                  |                     |                  |
                   |mg_tasks          | .. move tasks to..> |tasks             |
                   |                  |                     |                  |
                   |mg_preload_node + |                     |mg_preload_node + |
                   |                . |                     |                . |
                   |mg_node   +     . |                     |mg_node  +      . |
                   +------------------+                     +------------------+
                              *     .                                 *      .
                              *     .                                 *      .
    cgroup_mgctx              *     .                                 *      .
    +-----------------------+ *     .                                 *      .
    |                       | *     .                                 *      .
    |preloaded_src_csets    |=========                                *      .
    |preloaded_dst_csets    |==================================================
    |    (struct list_head) | *                                       *
    |                       | *                                       *
    |                       | *                                       *
    |tset                   | *                                       *
    |    (cgroup_taskset)   | *                                       *
    |    +------------------+ *                                       *
    |    |src_csets         |=====                                    *
    |    |dst_csets         |==================================================
    |    |(struct list_head)|
    +----+------------------+
```

  * 函数的目的是将task->cgroups从src_cset设置到dst_cset
  * mg_src_cgrp/mg_dst_cgrp分别记录了所要移动的源和目的cgroup
  * mg_dst_cset保存了task将要被移动到的css_set
  * task也将从src_cset->mg_tasks链表移动到dst_cset->tasks链表

这个函数不仅可以移动一个task，还能移动一个threadgroup。在这个情况下：

  * 那么将会把多个task->cgroups从各自的src_cset设置到给自的dst_cset
  * cgroup_mgctx是一个辅助结构，用来保存所有将会涉及到的css_set
  * 其中preloaded_src_csets/preloaded_dst_csets用于保存涉及到的源/目的css_set
  * tset用来给真实迁移task时使用

[1]: https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v1/cgroups.html
