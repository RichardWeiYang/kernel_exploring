内核中发生死锁的情况还是常常会有的，我们这次来看看一个真实发生死锁的情况，看看别人是怎么分析的。

[这是][1]一个A-B-B-A，锁顺序不统一导致的死锁。

首先，可以看到涉及的是两个进程。

```
INFO: task syz.5.48:5749 blocked for more than 143 seconds.
      Not tainted syzkaller #0

...

INFO: task syz.5.48:5754 blocked for more than 143 seconds.
      Not tainted syzkaller #0

```

也就是说进程5749和进程5754被阻塞了。

然后从下面看这两个进程都拿到了哪些锁。

```
1 lock held by syz.5.48/5749:
 #0: ffff888011d3f618 (&hugetlbfs_i_mmap_rwsem_key){++++}-{4:4}, at: i_mmap_lock_read include/linux/fs.h:568 [inline]
 #0: ffff888011d3f618 (&hugetlbfs_i_mmap_rwsem_key){++++}-{4:4}, at: __rmap_walk_file+0x227/0x620 mm/rmap.c:2905

3 locks held by syz.5.48/5754:
 ...
 #2: ffff888011d3f618 (&hugetlbfs_i_mmap_rwsem_key){++++}-{4:4}, at: i_mmap_lock_write include/linux/fs.h:548 [inline]
 #2: ffff888011d3f618 (&hugetlbfs_i_mmap_rwsem_key){++++}-{4:4}, at: hugetlbfs_punch_hole fs/hugetlbfs/inode.c:691 [inline]
 #2: ffff888011d3f618 (&hugetlbfs_i_mmap_rwsem_key){++++}-{4:4}, at: hugetlbfs_fallocate+0x4b5/0x1100 fs/hugetlbfs/inode.c:741
```

从这里看到，这两个进程都会去拿mmap_lock这个锁。其实这个时候是说两进程都拿到了这个锁，那么意味着他们有人还要拿另一个锁，但是这个锁被另一个拿了。

这时候要再回到call trace。

```
task:syz.5.48        state:D stack:26920 pid:5754  tgid:5747  ppid:5477   task_flags:0x400040 flags:0x00080002
Call Trace:
 <TASK>
 context_switch kernel/sched/core.c:5325 [inline]
 __schedule+0x1798/0x4cc0 kernel/sched/core.c:6929
 __schedule_loop kernel/sched/core.c:7011 [inline]
 schedule+0x165/0x360 kernel/sched/core.c:7026
 io_schedule+0x80/0xd0 kernel/sched/core.c:7871
 folio_wait_bit_common+0x6b0/0xb80 mm/filemap.c:1330
 __folio_lock mm/filemap.c:1706 [inline]
 folio_lock include/linux/pagemap.h:1141 [inline]
 __filemap_get_folio+0x139/0xaf0 mm/filemap.c:1960
 filemap_lock_folio include/linux/pagemap.h:820 [inline]
 filemap_lock_hugetlb_folio include/linux/hugetlb.h:814 [inline]
 ...
```

可以看到，进程5754卡在了folio_lock。那么这个时候就有可能是进程5749已经锁了folio，这样导致5754没法拿到。

这个时候再回到代码去看看是不是有这个可能。

根据lance的分析，确实有这个可能。

```
# Task (5749)
migrate_pages()
  -> migrate_hugetlbs()
    -> unmap_and_move_huge_page()     <- Takes folio_lock!
      -> remove_migration_ptes()
        -> __rmap_walk_file()
          -> i_mmap_lock_read()       <- Waits for i_mmap_rwsem(read lock)!

# Task (5754)
hugetlbfs_fallocate()
  -> hugetlbfs_punch_hole()           <- Takes i_mmap_rwsem(write lock)!
    -> hugetlbfs_zero_partial_page()
     -> filemap_lock_hugetlb_folio()
      -> filemap_lock_folio()
        -> __filemap_get_folio        <- Waits for folio_lock!
```

好了，至此我们学习了一个死锁案例，下次碰到这样的问题就有点思路了。

[1]: lkml.kernel.org/r/68e9715a.050a0220.1186a4.000d.GAE@google.com
