
# RCU做到了什么

> RCU supports concurrency between a single updater and multiple readers.

在RCU之前，我们只能做到的是：

  * 通过普通锁，同一时间只有一个人能访问，包括读和写
  * 通过读写锁，同一时间可以有多个人读，但是此时不能有写操作

而RCU的出现，能够做到在有一个人写的同时，可以有多个人读。

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

# 参考资料

[内核文档--RCU概念][1]

[1]: https://docs.kernel.org/RCU/index.html
[2]: https://lwn.net/Articles/262464/
