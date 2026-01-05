当THP功能打开后，内核会启动一个专门的线程khugepaged，将原本不是大页的页表合并成大页。


# 加入扫描队列

khugepaged作为一个线程会不停扫描自己的队列，看是否有需要合并的内存。我们先来看看这个队列的结构。

```
    khugepaged_scan
    +---------------------------+
    |address                    |  当前扫描到的虚拟地址
    |   (unsigned long)         |
    |mm_slot                    |  当前正在扫描的进程信息
    |   (khugepaged_mm_slot *)  |
    |   +-----------------------+
    |   |slot                   |
    |   |   (struct mm_slot)    |
    |   |   +-------------------+
    |   |   |mm                 |  指向进程的mm_struct
    |   |   |                   |
    |   |   |mm_node            |  添加到khugepaged_scan.mm_head的节点
    |   |   | (struct list_head)|
    |   |   |hash               |  添加到mm_slots_hash表的节点
    |   |   | (hlist_node)      |
    |   +-----------------------+
    |                           |
    |mm_head                    |  一个用于存放khugepaged_mm_slot.slot的链表
    |   (struct list_head)      |
    |                           |
    +---------------------------+

    mm_slots_hash
    +---------------------------+
    |                           |  一个用于存放khugepaged_mm_slot.slot的hash表
    +---------------------------+

```

将需要扫描的进程加入队列的函数是khugepaged_enter_vma()。每次都会分配一个khugepaged_mm_slot结构，然后将这个结构添加到khugepaged_scan.mm_head和mm_slots_hash中。

# 扫描&合并

为了有一个整体的感觉，我们把中间重要的流程抽取出来。

首先是扫描的过程：

```
collapse_scan_mm_slot()                     扫描某一个进程
    collapse_single_pmd()                   扫描进程中某一段PMD范围
        collapse_scan_pmd()                 扫描进程中一段匿名PMD范围
	    // 扫描PMD中的pte，符合条件后继续
            collapse_huge_page()            尝试合并
```

其次是合并的过程：

```
collapse_huge_page()
    alloc_charge_folio()                    分配好替换内存
    __collapse_huge_page_swapin()           如果有swap entry，先把内存调进来
    __collapse_huge_page_isolate()
    __collapse_huge_page_copy()             将内容复制到分配出的替换内存中
```

## 进程级扫描

khugepaged的扫描可以分为两个级别

  * 进程级
  * vma级

首先我们来看进程级扫描，collapse_scan_mm_slot()。这个级别主要是在遍历khugepaged.mm_head。

初始的时候，khugepaged_scan.mm_slot是空的，所以它会从khugepaged.mm_head上取下一个，写到khugepaged.mm_slot，开始做vma级扫描。

当这个进程的所有vma都扫描完后，则从khugepaged.mm_head上取出下一个，循环往复。

## vma级扫描

vma级扫描也是在函数collapse_scan_mm_slot()里，不过是在for_each_vma()这个循环中按照每2M的大小逐个扫描。


## PMD级扫描

最后就是按照PMD对齐进行扫描collapse_single_pmd()，其中又分了匿名页和文件页。

这次我们只看匿名页collapse_scan_pmd()。

整个扫描过程就是对PMD这块区间的每个pte进行检查，对于不符合要求的就跳过。

## 合并

到这里，我们已经找到了合适的目标，接下来就是将这部分pmd用一个thp来代替，这个工作交给了collapse_huge_page()。

合并的过程又可以分为

  * swapin: __collapse_huge_page_swapin()
  * isolate: __collapse_huge_page_isolate()
  * copy: __collapse_huge_page_copy()

先把swapout的页面拿进来，然后隔离，最后把内容拷贝到新分配的大页。如果成功了，将原先的页面释放，否则恢复原先页面。

# TRACE事件

在代码中增加了trace用来观察khugepaged的行为，便于后期优化。

## 定义和记录

首先在文件include/trace/events/huge_memory.h中定义了TRACE_EVENT。

比如:

```
TRACE_EVENT(mm_khugepaged_scan_pmd,
		...

```

然后在代码中需要的位置，加上

```
	trace_mm_khugepaged_scan_pmd(mm, folio, referenced,
				     none_or_zero, result, unmapped);
```

这里传入的参数都在TRACE_EVENT定义的时候指定。

## 观察event

代码是加上了，我们来看看怎么观测。对应的方式有很多，我们来看看最原始的一种。

### 位置

对应的trace的sysfs在/sys/kernel/tracing/events/huge_memory/。拿上面的 TRACE_EVENT(mm_khugepaged_scan_pmd, ...)来说，就是在

```
/sys/kernel/tracing/events/huge_memory/mm_khugepaged_scan_pmd
```

这个目录下有多个文件：

```
# ls /sys/kernel/tracing/events/huge_memory/mm_khugepaged_scan_pmd/
enable  filter  format  hist  id  inject  trigger
```

目前了解下来，要是用trace来观测，需要这么几个步骤。

  - 了解格式
  - 定制输出
  - 使能事件
  - 观察输出

### 了解格式

每一个trace事件的输出是不同的，了解这个输出是为了后面定制化事件的输出。

输出的格式在format文件中。


```
# cat format
name: mm_khugepaged_scan_pmd
ID: 689
format:
	field:unsigned short common_type;	offset:0;	size:2;	signed:0;
	field:unsigned char common_flags;	offset:2;	size:1;	signed:0;
	field:unsigned char common_preempt_count;	offset:3;	size:1;	signed:0;
	field:int common_pid;	offset:4;	size:4;	signed:1;

	field:struct mm_struct * mm;	offset:8;	size:8;	signed:0;
	field:unsigned long pfn;	offset:16;	size:8;	signed:0;
	field:bool writable;	offset:24;	size:1;	signed:0;
	field:int referenced;	offset:28;	size:4;	signed:1;
	field:int none_or_zero;	offset:32;	size:4;	signed:1;
	field:int status;	offset:36;	size:4;	signed:1;
	field:int unmapped;	offset:40;	size:4;	signed:1;
```

这些其实在TRACE_EVENT定义里就能看到，不过为了用户方便，还是在用户态暴露了出来。

### 定制输出

我们看到上面输出这么多，不一定每次都需要。所以我们可以通过trigger文件定制化事件的输出。

比如mm_khugepaged_scan_pmd这个事件，查看result情况并按照触发次数排序。就可以这么设置：

```
echo 'hist:keys=status:sort=hitcount' > trigger
```

PS: 这只是最基本的，感觉要还有很多不同的用法。

### 使能事件

这个比较简单

```
echo 1 > enable
```

### 观察输出

事件采集到了hist文件，我们看下刚才这么设置后的输出。

```
# cat hist
# event histogram
#
# trigger info: hist:keys=status:vals=hitcount:sort=hitcount:size=2048 [active]
#

{ status:          7 } hitcount:         19
{ status:          6 } hitcount:         21
{ status:          5 } hitcount:         23
{ status:          1 } hitcount:         37
{ status:          3 } hitcount:         46
{ status:          4 } hitcount:        388

Totals:
    Hits: 534
    Entries: 6
    Dropped: 0
```

从这个结果看，这段事件内mm_khugepaged_scan_pmd的结果中，返回4的是最多的。

# 测试

针对大页合并，内核中提供了一个用户态的测试程序。

tools/testing/selftests/mm/khugepaged.c

这算是[selftests][1]的一个例子。


[1]: /mm/tests/01_functional_test.md
