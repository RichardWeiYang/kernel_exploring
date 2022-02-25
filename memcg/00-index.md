memcg在当前云计算场景下被广泛应用，其目标是限定用户对主机内存的使用。本章我们将对memcg的工作原理做探究。

首先我们来简单看一下[memcg初始化][1]的过程。因为memcg是cgroup的子系统，所以更多的细节可以看[cgroup][2]。

既然memcg是用来限制用户对内存使用的，那么自然我们要了解如何[限制memcg大小][3]和[对memcg记账][4]


[1]: /memcg/01-init_overview.md
[2]: /cgroup/00-index.md
[3]: /memcg/02-set_memcg_limit.md
[4]: /memcg/03-charge_memcg.md
