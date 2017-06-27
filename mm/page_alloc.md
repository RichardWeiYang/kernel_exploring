memblock作为系统启动初期的内存管理系统，在系统启动后就显得不是那么灵活了。在这个时候，内核就提出了页分配器来管理物理内存。而我们在Node->Zone->Page这篇中看到的这个层次结构，就是为了页分配器做的准备。

# 页从哪来

一路走来，我们回顾一下内存信息是怎么收集上来的

```
    BIOS -> e820 -> memblock
```

那现在就要看系统是怎么从memblock的信息中，组合成page的信息的。

众所周知，在内核中我们使用page结构体来表示物理页。在SPARSEMEM这篇中我们也对page结构体的存储位置做了探索。那在最开始的时候，这个结构体是空的。那什么时候系统把物理实际存在的页面和这个page结构体对应上呢？系统是什么时候确认系统到底是有那些page可以使用的呢？如何把page结构体保存到相应的Zone结构体中的呢？

这个过程就在free_all_bootmem()函数中。

```
    start_kernel()
        mm_init()
            mem_init()
                free_all_bootmem()
```

在这个过程中你可以看到熟悉的for_each_free_mem_range()。对了，这就是遍历memory_block的内存信息，来填充page结构体的过程了。

# 释放

通常我们都是先讲分配，再讲释放的。但是这次要到过来。为什么呢，因为在free_all_bootmem()中就是调用了内存释放的方法来初始化page结构的。所以还是先来看看释放吧。

我们通常使用的函数是free_pages()。但是最后都调用到了函数__free_one_page()。这个函数其实比较简单，就是算的东西稍微有点绕。我们都知道页分配器中使用的是伙伴系统，也就是相邻的两个页是一对儿。所以在释放的时候，会去判断自己的伙伴是不是也是空闲的。如果是的话，那就做合并。且继续判断。

关于伙伴查找的算法，后面详细讲。有兴趣的可以仔细看。

那最后释放到哪里了呢？

对了，就是zone->free_area[order].free_list上了。

# 分配

页分配的核心函数是__alloc_pages_nodemask()。正如函数的注释上写的"This is the 'heart' of the zoned buddy allocator."。

在[Node->Zone->Page][2]中我们已经看到了，内存被分为了node，zone来管理。这么管理的目的也就是为了在分配的时候，能能够找到合适的内存。所以在分配的时候，就是按照优先级搜索node和zone，如果找到匹配的zone则在该zone的free_area链表中取下一个。

和释放内存对称，分配的时候也可能会分配到高阶的page。如果发生这种情况，则会将高阶部分放回到对应的free_area中。

# MAX_ORDER不是MAX_ORDER

这个标题有点绕吧。不过你看完了就知道确实就是这个意思。

我们知道zone->free_area[MAX_ORDER]上排列着指定大小的空闲页。一般情况下MAX_ORDER定义为11。但是你知道吗，这个MAX_ORDER的值并不表示内核管理空闲页最大单位是2^MAX_ORDER。

一个数组的个数和数组的下标之间相差为1，所以free_area表示的是2^0到2^10单位的空闲页。

记住了哈，页分配器中管理的最大空闲页是2^(MAX_ORDER -1)的大小。

# 伙伴系统

## 你不是一个人

内核中的buddy system可谓是久仰大名，如雷贯耳。这年头找对象不容易，看看人内核里面是怎么找的，咱也好好学学，取取经～

## 敬个礼，握握手，你是我的好朋友

好了，说正经的，人是怎么找的呢？

```c

__free_one_page()
...
	while (order < max_order - 1) {
		buddy_idx = __find_buddy_index(page_idx, order);
		buddy = page + (buddy_idx - page_idx);
		if (!page_is_buddy(page, buddy, order))
			goto done_merging;
		/*
		 * Our buddy is free or it is CONFIG_DEBUG_PAGEALLOC guard page,
		 * merge with it and move up one order.
		 */
		if (page_is_guard(buddy)) {
			clear_page_guard(zone, buddy, order, migratetype);
		} else {
			list_del(&buddy->lru);
			zone->free_area[order].nr_free--;
			rmv_page_order(buddy);
		}
		combined_idx = buddy_idx & page_idx;
		page = page + (combined_idx - page_idx);
		page_idx = combined_idx;
		order++;
	}
...
```

其中重要的就是__find_buddy_index()。
```c
/*
 * Locate the struct page for both the matching buddy in our
 * pair (buddy1) and the combined O(n+1) page they form (page).
 *
 * 1) Any buddy B1 will have an order O twin B2 which satisfies
 * the following equation:
 *     B2 = B1 ^ (1 << O)
 * For example, if the starting buddy (buddy2) is #8 its order
 * 1 buddy is #10:
 *     B2 = 8 ^ (1 << 1) = 8 ^ 2 = 10
 *
 * 2) Any buddy B will have an order O+1 parent P which
 * satisfies the following equation:
 *     P = B & ~(1 << O)
 *
 * Assumption: *_mem_map is contiguous at least up to MAX_ORDER
 */
static inline unsigned long
__find_buddy_index(unsigned long page_idx, unsigned int order)
{
	return page_idx ^ (1 << order);
}
```

人注释写得真好，还有个例子给你看。我这个人有个毛病，一旦看懂了代码就老兴奋了。忍不住多说两句，估计人真正的大牛对这种代码了如指掌，根本没兴趣了。

我个人看完这个代码有两点体会：
* 使用某个位置上的数做异或就像是一个开关，原来有的就清，没的就置位
*  两个小伙伴的上一级伙伴，是这个位为零的那个

## 举个栗子

     +--------+--------+--------+--------+
     |8       |9       |10      |11      |
     +--------+--------+--------+--------+
                                       
     [======  2  =====] [=====  2  ======]
                                        
     [###############  4  ###############] 

举个栗子， 从上面这个图看出，O为1的时候，（8，9）为一组，（10，11）为一组。 

 ```
          8   ^ 2 =  0b 1000 ^ 0b 0010 = 0b 1010 = 10
          10 ^ 2 =  0b 1010 ^ 0b 0010 = 0b 1000 = 8
 ```
可以看到，异或的结果就是order 2的那一位去反。很有意思。恩，是不是可以理解为男生找女生，女生找男生呢？ （如果你硬要找一样的，我就没办法了）

那我们再来看如何找上一级的伙伴。从图上直观来看， (8,9) (9,10)的上一级就是数量为4的组。那他的首页面index就是8。

好了看看代码实现的效果
 ```
          8   & ~2 =  0b 1000 & 0b 1101 = 0b 1000 = 8
          10 & ~2 =  0b 1010 & 0b 1101 = 0b 1000 = 8
 ```
直观的效果就是大家都将order 2的那一位清除。哇哦，so easy.

大家注意一下实际代码中的一句

```c
		combined_idx = buddy_idx & page_idx;
		page_idx = combined_idx;
```

在循环中，page_idx将作为下一次循环要去寻找伙伴的索引。那你想到了啥？
对了，正是因为自己的上一级伙伴是把order 2的那一位清空，所以自己和自己小伙伴做与运算就得到了上一级～

好了，我想你知道内核页分配中伙伴是怎么找的了～

到这里，就到这里，休息休息~


[1]: http://blog.csdn.net/richardysteven/article/details/52332040
[2]: http://blog.csdn.net/richardysteven/article/details/68482849