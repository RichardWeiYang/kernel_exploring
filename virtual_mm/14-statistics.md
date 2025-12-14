当进程执行内存相关操作时，如分配大页，内核会记录下相关动作。这些数据作为进程生命周期的一部分存在。

# mm->rss_stat[]

## 位置

首先我们来看看mm->rss_stat这个统计数据，在mm_struct中有一个rss_stat[]数组。

## 类型

这个数组一共就这么些类型。

```
enum {
	MM_FILEPAGES,	/* Resident file mapping pages */
	MM_ANONPAGES,	/* Resident anonymous pages */
	MM_SWAPENTS,	/* Anonymous swap entries */
	MM_SHMEMPAGES,	/* Resident shared memory pages */
	NR_MM_COUNTERS
};
```

## 操作

对操作的方法是

修改类:

  * add_mm_counter()
  * inc_mm_counter()
  * dec_mm_counter()

读取类:

  * get_mm_counter_sum()

这个看上去是统计进程中各种类型内存的数量，以base page为单位。

比如在do_anonymous_page()中，如果分配成功了匿名页，则会 add_mm_counter(vma->vm_mm, MM_ANONPAGES, nr_pages);来增加对应的计数。

而在do_swap_page()中，如果是交换进页面，则会

```
	add_mm_counter(vma->vm_mm, MM_ANONPAGES, nr_pages);
	add_mm_counter(vma->vm_mm, MM_SWAPENTS, -nr_pages);
```

增加匿名页计数，同时减少swap页计数。

## 展示

这里统计的值会在文件[/proc/self/status][1]中展示。

其中的RssAnon, RssFile, RssShmem和VmSwap。

# vm_event_state

## 位置

这是一个全局的per_cpu数组。

```
struct vm_event_state {
	unsigned long event[NR_VM_EVENT_ITEMS];
};

DECLARE_PER_CPU(struct vm_event_state, vm_event_states);
```

此外，这个还是一个内核配置项。只有配置了CONFIG_VM_EVENT_COUNTERS才会记录相关的事件信息。

## 类型

这个种类就多很多了， 详见vm_event_item，目测有120+种。

## 操作

这个事件计数的操作比较简单：

变更：

  - _count_vm_event()
  - count_vm_event()
  - _count_vm_events()
  - count_vm_events()

读取：

  - all_vm_enents()

## 观察

这个计数是可以观察的，也就是在我们熟知的[vmstat][3]文件。

不过vmstat文件中，不仅包含了vm_event_states的信息。所以事件记录是从pgpgin字段开始的。

# pgdat->vm_stat[]/vm_node_stat[]

## 位置

看了下，发现一个有意思的东四，有两个一模一样的统计数组。

  * pgdat->vm_stat[]
  * vm_node_stat[]

两个都是原子变量，只不过一个在pgdat中，一个是全局的。

```
atomic_long_t vm_node_stat[NR_VM_NODE_STAT_ITEMS] __cacheline_aligned_in_smp;
```

所以可以看到，最核心的变量更新函数 node_page_state_add()是对这两个变量同时更新的。

```
static inline void node_page_state_add(long x, struct pglist_data *pgdat,
				 enum node_stat_item item)
{
	atomic_long_add(x, &pgdat->vm_stat[item]);
	atomic_long_add(x, &vm_node_stat[item]);
}
```

而且这个XXX_add()名字好迷惑，实际上只要传入的参数是负数，就是减法操作。

## 类型

一个枚举型数组，里面约有四十左右类型。

包括了active_anon, active_file等。

## 操作

目前看到操作这个数组的方式，除了直接通过原子操作atomic_long_read()之外，有下面几个封装。

修改类:

  - 以pgdate为参数,最核心的API，其余都是调用这个
      - node_page_state_add(pgdat, )
      - __inc_node_state(pgdat, )
      - __dec_node_state(pgdat, )
      - __mod_node_page_state(pgdat, )
      - mod_node_page_state(pgdat, ), 可能依赖mod_node_state()
  - 以page为参数
      - __inc_node_page_state(page, )
      - inc_node_page_state(page, )
      - __dec_node_page_state(page, )
      - dec_node_page_state(page, )
  - 以folio为参数
      - __node_stat_mod_folio(folio, )
      - node_stat_mod_folio(folio, )
      - __node_stat_add_folio(folio, )
      - node_stat_add_folio(folio, )
      - __node_stat_sub_folio(folio, )
      - node_stat_sub_folio(folio, )
      - __folio_mod_stat(folio, )

大致看了下，有下划线前缀的是不关中断的版本。偷懒的讲，不确定的时候就有没有下划线的版本比较保险。

基本上以page为参数的API应该是要退出历史舞台了，取而代之的是以folio为参数的。

但是统计数据中有些是和内存页无关的，比如PGPROMOTE_CANDIDATE，就还是用以pgdat为参数的API。

读取类:

  * global_node_page_state()
  * global_node_page_state_pages()

这两个蛮奇怪的，global_node_page_state()调用的是global_node_page_state_pages()。而前者会在某条件下warn一次。

## 展示

在[/proc/meminfo][2]中会读取global_node_page_state()来返回当前系统相关信息。

# zone->vm_stat[]/vm_zone_stat[]

## 位置

和vm_node_stat[]一样，有两个一模一样的数组：一个是全局原子变量，一个是在zone里的变量

```
atomic_long_t vm_zone_stat[NR_VM_ZONE_STAT_ITEMS] __cacheline_aligned_in_smp;
```

修改的时候，两者会同时修改。

## 类型

一个枚举型数组，里面约有四十左右类型。

## 操作

修改类:

  - 以zone为参数
      - zone_page_state_add(zone, x)
      - __inc_zone_state(zone, )
      - __dec_zone_state(zone, )
      - mod_zone_page_state(), 可能依赖mod_zone_state(zone, )
  - 以page为参数
      - __inc_zone_page_state(page, )
      - inc_zone_page_state(page, )
      - __dec_zone_page_state(page, )
      - dec_zone_page_state(page, )
  - 以folio为参数
      - __zone_stat_mod_folio(folio, )
      - zone_stat_mod_folio(folio, )
      - __zone_stat_add_folio(folio, )
      - zone_stat_add_folio(folio, )
      - __zone_stat_sub_folio(folio, )
      - zone_stat_sub_folio(folio, )

有意思的是，folio的这些api中，目前只用了zone_stat_mod_folio()。

而page的api，目前也只有zsmalloc在用了。

读取类:

  * global_zone_page_state()

## 展示

在[/proc/vmstat][3]中会读取global_zone_page_state()来返回当前系统相关信息。

[1]: /mm/statistics/06-status.md
[2]: /mm/statistics/07-meminfo.md
[3]: /mm/statistics/08-vmstat.md
