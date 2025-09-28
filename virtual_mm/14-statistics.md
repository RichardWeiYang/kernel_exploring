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

```
add_mm_counter()
inc_mm_counter()
dec_mm_counter()
```

这个看上去是统计进程中各种类型内存的数量，以base page为单位。

比如在do_anonymous_page()中，如果分配成功了匿名页，则会 add_mm_counter(vma->vm_mm, MM_ANONPAGES, nr_pages);来增加对应的计数。

而在do_swap_page()中，如果是交换进页面，则会

```
	add_mm_counter(vma->vm_mm, MM_ANONPAGES, nr_pages);
	add_mm_counter(vma->vm_mm, MM_SWAPENTS, -nr_pages);
```

增加匿名页计数，同时减少swap页计数。
