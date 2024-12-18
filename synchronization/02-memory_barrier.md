这部分的内容，主要来自于[1]的学习笔记。

# 为什么需要内存屏障

因为下面几个原因，导致需要内存屏障：

  * 编译器优化代码
  * CPU乱序执行
  * 内存一致性
  * 内存预读取

PS: 在[1]的GUARANTEES部分，提到了编译器和CPU会保证顺序的情况--访问的变量有前后依赖的情况。

而内存屏障提供了一种方法，让代码的执行顺序能按照我们写的来。

PS: 实际上应该是内存屏障和编译屏障一起提供保障。

# 内存屏障的种类

通常我们看到的种类有：

  * Write memory barrier
  * Read memory barrier
  * General memory barrier

这几个的定义可以用一句话概括：

> all the LOAD/STORE operations specified before the barrier will appear to **happen before** all the LOAD/STORE operations specified after the barrier with respect to the other components of the system.

以及隐含变体：

  * ACQUIRE operation
  * RELEASE operation

这两个有点特别，不是很懂。

# Address Dependency Barrier(Historical)

我们先看一个例子：

```
        CPU 1                 CPU 2
        ===============       ===============
        { A == 1, B == 2, C == 3, P == &A, Q == &C }
        B = 4;
        <write barrier>
        WRITE_ONCE(P, &B);
                              Q = READ_ONCE_OLD(P);
                              D = *Q;
```

这里面有个很神奇的情况，就是从CPU2的角度看P已经赋值为&B，但是B还是2。（这种情况会发生在split cache的机器上）

具体解决方法是使用READ_ONCE()，因为当前的READ_ONCE()里隐藏了address dependency。

```
        CPU 1                 CPU 2
        ===============       ===============
        { A == 1, B == 2, C == 3, P == &A, Q == &C }
        B = 4;
        <write barrier>
        WRITE_ONCE(P, &B);
                              Q = READ_ONCE(P);
                              <implicit address-dependency barrier>
                              D = *Q;
```

这里需要关注的是，如果没有隐藏的address
dependency，会影响rcu的功能。我们可以把P看作一个全局指针，B是一个新的版本。当CPU2认为P已经更新到B这个版本时，如果看到的B里内容不是最新的，那就有问题了。

# Control Dependency

这部分主要是因为编译器会对if这样的判断语句做优化，导致代码不按照我们的预期执行。

从形式上看，又可以分成两类：

  * load-load control dependency
  * load-store control dependency

## load-load

比如这样的情况

```
        q = READ_ONCE(a);
        <implicit address-dependency barrier>
        if (q) {
                /* BUG: No address dependency!!! */
                p = READ_ONCE(b);
        }
```

两个READ_ONCE之间没有地址上的依赖，一个是从a读，一个是从b读，所以CPU可以打乱两者的顺序。
所以正确的写法是

```
        q = READ_ONCE(a);
        if (q) {
                <read barrier>
                p = READ_ONCE(b);
        }
```

PS: 如果是单线程纯内存访问，不加barrier可能也没问题。最后还是要判断q是不是非空，才会赋值到p。但是如果是访问设备寄存器的话，就必须加barrier了。

## load-store

写的情况稍微好些，因为之间有一定的地址依赖：

```
        q = READ_ONCE(a);
        if (q) {
                WRITE_ONCE(b, 1);
        }
```

PS: 其实我好像没有看出来有依赖，但意思就是能。而且必须是先读后写。

另外重要的是READ_ONCE和WRITE_ONCE是必须要的。

后面还有几个编译器能预测出if结果的例子，这里就不展开了。

# SMP Barrier Pairing

这一部分主要关注的是多个CPU访问同一段内存的情况，这个问题是由内存一致性引入的。

为了解决这个问题，通常需要**内存屏障成对出现**。

例如：

```
        CPU 1                   CPU 2
        ======================= =======================
                { A = 0, B = 9 }
        STORE A=1
        <write barrier>
        STORE B=2
                                LOAD B
                                <read barrier>
                                LOAD A
```

CPU1/CPU2中各自要加上屏障，才能保证CPU2上读到B==2后，A等于1。

# 内核中显式屏障

这个又分成两种：

  * compiler barrier
  * cpu memory barrier

## Compiler Barrier

  * barrier()
  * READ_ONCE()/WRITE_ONCE()

不过READ_ONCE()/WRITE_ONCE()还有点cpu memory barrier的作用。

比如这个例子：

```
        a[0] = READ_ONCE(x);
        a[1] = READ_ONCE(x);
```

说是可以防止a[1]的值不会比a[0]的旧，而仅仅是编译器屏障，是不能保证cpu不会乱序执行的。

另外还有个例子：

作者说下面的代码

```
        while (tmp = a)
                do_something_with(tmp);
```

会被编译器优化成

```
        if (tmp = a)
                for (;;)
                        do_something_with(tmp);
```

所以以后像while/if这种判断里面，最好是不要做赋值操作了。

或者也会因为寄存器不够用，将上面的代码优化成：

```
        while (a)
                do_something_with(a);
```

这样如果有另一个线程在do_something_with()前更改了a，那就不是我们想要的行为了。

## CPU memory barrier

内核中其中基本的内存屏障：

|Type                 |Mandatory             |SMP Conditional  |
|---------------------|----------------------|-----------------|
|General              |mb()                  |smp_mb()         |
|Write                |wmb()                 |smp_wmb()        |
|Read                 |rmb()                 |smp_rmb()        |
|Address Dependency   |                      |READ_ONCE()      |

除了Address Dependency, 都隐含了编译器屏障。

# 内核中隐含屏障

除了显式内存屏障，内核中有些接口隐含调用了内存屏障：

  * 锁
  * 睡眠/唤醒
  * 调度

# 哪里需要用到内存屏障？

代码执行顺序重排，在下面几个情况会产生问题：

  * interprocessor interaction
  * atomic operation
  * accessing device
  * interrupt

# 参考资料

[Memory Barriers][1]

[1]: https://docs.kernel.org/core-api/wrappers/memory-barriers.html
