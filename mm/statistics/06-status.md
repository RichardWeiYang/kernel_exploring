/proc/pid/status这个文件展示的是整个进程的状态，其中有部分是和内存相关的。

# 代码

展示相关代码在fs/proc/base.c的proc_pid_status中。

而统计的代码可以看[统计数据][1]中mm->rss_stat[]部分。

# 格式

```
Name:	cat
Umask:	0002
State:	R (running)
Tgid:	5880
Ngid:	0
Pid:	5880
PPid:	3010
TracerPid:	0
Uid:	1000	1000	1000	1000
Gid:	1000	1000	1000	1000
FDSize:	256
Groups:	4 24 27 30 46 122 135 136 1000 
NStgid:	5880
NSpid:	5880
NSpgid:	5880
NSsid:	3010
Kthread:	0
VmPeak:	   11604 kB
VmSize:	   11604 kB
VmLck:	       0 kB
VmPin:	       0 kB
VmHWM:	    2176 kB
VmRSS:	    2176 kB
RssAnon:	       0 kB
RssFile:	    2176 kB
RssShmem:	       0 kB
VmData:	     360 kB
VmStk:	     132 kB
VmExe:	      16 kB
VmLib:	    1796 kB
VmPTE:	      64 kB
VmSwap:	       0 kB
HugetlbPages:	       0 kB
CoreDumping:	0
THP_enabled:	1
untag_mask:	0xffffffffffffffff
Threads:	1
SigQ:	0/43285
SigPnd:	0000000000000000
ShdPnd:	0000000000000000
SigBlk:	0000000000000000
SigIgn:	0000000000000000
SigCgt:	0000000000000000
CapInh:	0000000000000000
CapPrm:	0000000000000000
CapEff:	0000000000000000
CapBnd:	000001ffffffffff
CapAmb:	0000000000000000
NoNewPrivs:	0
Seccomp:	0
Seccomp_filters:	0
Speculation_Store_Bypass:	vulnerable
SpeculationIndirectBranch:	always enabled
Cpus_allowed:	ff
Cpus_allowed_list:	0-7
Mems_allowed:	00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000001
Mems_allowed_list:	0
voluntary_ctxt_switches:	0
nonvoluntary_ctxt_switches:	0
x86_Thread_features:	
x86_Thread_features_locked:
```

这是整个输出，其中Vm开头的是和内存相关的。

# 使用

[1]: /virtual_mm/14-statistics.md
