/proc/pid/maps和/proc/pid/smaps展示的是以vma为单位的信息。smaps是maps更详细的版本。

# 代码

每个进程都有，自然也是在tid_base_stuff[]数组中。分别是proc_pid_maps_operations和proc_pid_smaps_operations。

proc_pid_maps_operations最后展示的时候调用的是show_map。

proc_pid_smaps_operations最后展示的时候调用的是show_smap。

# 格式

文件中，对应进程的每个vma，都有类似下面一段输出。

```
ffffffffff600000-ffffffffff601000 --xp 00000000 00:00 0                  [vsyscall]
Size:                  4 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
Rss:                   0 kB
Pss:                   0 kB
Pss_Dirty:             0 kB
Shared_Clean:          0 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:         0 kB
Referenced:            0 kB
Anonymous:             0 kB
KSM:                   0 kB
LazyFree:              0 kB
AnonHugePages:         0 kB
ShmemPmdMapped:        0 kB
FilePmdMapped:         0 kB
Shared_Hugetlb:        0 kB
Private_Hugetlb:       0 kB
Swap:                  0 kB
SwapPss:               0 kB
Locked:                0 kB
THPeligible:           0
VmFlags: ex
```

# 用法

比如在toos/testing/selftests/mm/split_huge_page_test.c中，通过比较AnonHugePages的值来判断是否有大页。
