该文件保存了整个系统的内存使用情况。

# 代码

内核中fs/proc/meminfo.c是用来创建这个文件的代码。其中主要也就是meminfo_proc_show这个函数。

# 格式

```
MemTotal:       11164052 kB         总内存
MemFree:         4713252 kB         空闲
MemAvailable:    8871388 kB         MemAvailable > MemTotal - MemFree
Buffers:          103928 kB
Cached:          4102152 kB
SwapCached:            0 kB
Active:          2949276 kB         Active = Active(anon) + Active(file)
Inactive:        2858716 kB
Active(anon):    1568216 kB
Inactive(anon):        0 kB
Active(file):    1381060 kB
Inactive(file):  2858716 kB
Unevictable:           0 kB
Mlocked:               0 kB
SwapTotal:       2097148 kB
SwapFree:        2097148 kB
Zswap:                 0 kB
Zswapped:              0 kB
Dirty:                56 kB
Writeback:             0 kB
AnonPages:       1601952 kB         AnonPages > Active(anon) + Inactive(anon)
Mapped:           645472 kB
Shmem:             98124 kB
KReclaimable:     233628 kB
Slab:             398252 kB
SReclaimable:     233628 kB
SUnreclaim:       164624 kB
KernelStack:       14688 kB
PageTables:        29616 kB
SecPageTables:         0 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:     7679172 kB
Committed_AS:    5980692 kB
VmallocTotal:   34359738367 kB
VmallocUsed:       33120 kB
VmallocChunk:          0 kB
Percpu:             6784 kB
HardwareCorrupted:     0 kB
AnonHugePages:         0 kB
ShmemHugePages:        0 kB
ShmemPmdMapped:        0 kB
FileHugePages:         0 kB
FilePmdMapped:         0 kB
Unaccepted:            0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:               0 kB
DirectMap4k:      128960 kB
DirectMap2M:    11354112 kB
```

其中MemAvailable > MemTotal - MemFree是因为有些内存是可以reclaim的。

但是AnonPages > Active(anon) + Inactive(anon)我不太理解。就算不一样，也应该是小于才对吧？

# 使用
