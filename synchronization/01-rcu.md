
# RCU做到了什么

> RCU supports concurrency between a single updater and multiple readers.

在RCU之前，我们只能做到的是：

  * 通过普通锁，同一时间只有一个人能访问，包括读和写
  * 通过读写锁，同一时间可以有多个人读，但是此时不能有写操作

而RCU的出现，能够做到在有一个人写的同时，可以有多个人读。同时这也意味着，写操作之间还需要其他锁机制做保护，RCU通常不会被单独使用。

当然，RCU对访问的数据也有一个适用范围。也就是对数据的实时性不敏感。

# RCU底层的三个机制

在[What is RCU][2]中，描述了RCU底层的三个机制：

  * Publish-Subscribe
  * Wait For Pre-Existing RCU Readers to Complete
  * Maintain Multiple Versions of Recent Updated Objects

## Publish-Subscribe

这个机制的底层原因是**编译器和cpu的乱序执行**。

为了让我们最终访问的数据如我们期待，而不会被改变顺序。内核提供了写/读两方面的API，分别称为Publish/Subscribe。

下面这个表格举例了内核中针对不同数据结构给出的相应API

|Category|Publish                                                    |Retract                   |Subscribe                 |
|--------|-----------------------------------------------------------|--------------------------|--------------------------|
|Pointer |rcu_assign_pointer()                                       |rcu_assign_pointer(, NULL)|rcu_dereference()         |
|List    |list_add_rcu()<br>list_add_tail_rcu()<br>list_replace_rcu()|list_del_rcu()            |list_for_each_entry_rcu() |
|Hlist   |hlist_add_after_rcu()<br>hlist_replace_rcu()               |hlist_del_rcu()           |hlist_for_each_entry_rcu()|

## Wait For Pre-Existing RCU Readers to Complete

本质上，RCU是一种等的功夫。但是厉害的是，**RCU并不记录具体跟踪的对象**。

> The great advantage of RCU is that it can wait for each of (say) 20,000 different things without having to explicitly track each and every one of them, and without having to worry about the performance degradation, scalability limitations, complex deadlock scenarios, and memory-leak hazards that are inherent in schemes using explicit tracking.

用rcu_read_lock()/rcu_read_unlock()原语界定的区域是RCU read-side critical section。这段区域可以包含任何代码，**除了 阻塞和睡眠(block or sleep)**。

文档[2]中描述了等待机制synchronize_rcu()的原理，就是让自己在所有cpu上都调度一遍。原因是：

> RCU Classic read-side critical sections delimited by rcu_read_lock() and rcu_read_unlock() are not permitted to block or sleep. Therefore, when a given CPU executes a context switch, we are guaranteed that any prior RCU read-side critical sections will have completed. 

这里的前提是RCU critical section中间的代码不能阻塞和睡眠，一旦调度发生，表明RCU read-side critical section一定执行完了。

所以当synchronize_rcu()返回，就可以清理相应的数据了。

## Maintain Multiple Versions of Recent Updated Objects

我觉得实际上就是通过上面的延迟销毁来做到的。

# 一个例子

在参考[3]中，有这么一个使用rcu的例子。

```
 1 struct el {
 2   struct list_head list;
 3   long key;
 4   spinlock_t mutex;
 5   int data;
 6   /* Other data fields */
 7 };
 8 spinlock_t listmutex;
 9 struct el head;

 1 int search(long key, int *result)
 2 {
 3   struct list_head *lp;
 4   struct el *p;
 5
 6   rcu_read_lock();
 7   list_for_each_entry_rcu(p, head, lp) {
 8     if (p->key == key) {
 9       *result = p->data;
10       rcu_read_unlock();
11       return 1;
12     }
13   }
14   rcu_read_unlock();
15   return 0;
16 }

 1 int delete(long key)
 2 {
 3   struct el *p;
 4
 5   spin_lock(&listmutex);
 6   list_for_each_entry(p, head, lp) {
 7     if (p->key == key) {
 8       list_del_rcu(&p->list);
 9       spin_unlock(&listmutex);
10       synchronize_rcu();
11       kfree(p);
12       return 1;
13     }
14   }
15   spin_unlock(&listmutex);
16   return 0;
17 }
```

其中在delete函数中有一个我开始不太理解的用法。

> 为什么这里遍历的时，用的是list_for_each_entry()，而不是list_for_each_entry_rcu()

我现在的理解是，因为update操作用spin_lock保证了此时没有别人变更链表，且锁本身保证了内存一致性。
所以现在用list_for_each_entry()看到的就是最新的版本，不会有老的版本。

但是此时删除为什么要用list_del_rcu()？后续的spin_unlock()难道不能保证内存被正确同步吗?
答：是我从函数名字上去理解了。list_del_rcu()的定义是：

```
static inline void list_del_rcu(struct list_head *entry)
{
	__list_del_entry(entry);
	entry->prev = LIST_POISON2;
}
```

也就是执行完后保留了entry->next，是为了list_for_each_entry()能够继续往后面去找。
而普通的list，在删除的时候是通过list_for_each_entry_safe()事先得到next。所以list_del()的时候能够直接清除next。

# 参考资料

[内核文档--RCU概念][1]
[RCU dereference][2]

[1]: https://docs.kernel.org/RCU/index.html
[2]: https://lwn.net/Articles/262464/
[3]: https://lwn.net/Articles/263130/
[4]: https://docs.kernel.org/RCU/rcu_dereference.html
