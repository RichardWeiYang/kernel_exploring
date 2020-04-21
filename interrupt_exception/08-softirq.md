软中断，softirq，经常听说，但是究竟是什么，怎么用其实我并不清楚。这不，最近看到有代码使用了软中断，为了能够进一步了解只好硬着头皮来看看源代码。

> 软中断是利用硬件中断的概念，用软件方式进行模拟，实现宏观上的异步执行效果。

那究竟是怎么做的呢？让我们一探究竟。

# 初始化

一切的一切总有个起头的，既然是抓瞎的状态，那就先抓住开始的部分。

```
start_kernel()
    softirq_init(void)
        open_softirq(TASKLET_SOFTIRQ, tasklet_action);
    	  open_softirq(HI_SOFTIRQ, tasklet_hi_action);
            softirq_vec[nr].action = action;
```

瞧，这其实啥都没干，就填写了一个数组的成员。那这个数组长什么样子呢？

```
      softirq_vec[NR_SOFTIRQS]
      +------------------------------------+
      |action                              |
      |  void (*)(struct softirq_action *) |
      +------------------------------------+
      |action                              |
      |  void (*)(struct softirq_action *) |
      +------------------------------------+
      |action                              |
      |  void (*)(struct softirq_action *) |
      +------------------------------------+
```

好吧，就这么一个光杆司令，每个数组就是一个毁掉函数的成员。真的是不知所云。

# 硬塞的__preemt_count

按照正常逻辑，现在我应该来讲什么时候softirq被调用的。但是呢，我发先有一个非常重要的概念（变量）对理解何时调用很关键，所以就硬插进来，模拟一个“软中断”。

这个变量叫__preemt_count。名字很简单，但是定义却藏着玄机。

```
DEFINE_PER_CPU(int, __preempt_count) = INIT_PREEMPT_COUNT;
```

乍一看就是一个32位的整型，但是你再往里看其实这个整型被切几个小块，没块有自己的含义。

```
            PREEMPT_MASK:	0x000000ff
            SOFTIRQ_MASK:	0x0000ff00
            HARDIRQ_MASK:	0x000f0000
                NMI_MASK:	0x00100000
    PREEMPT_NEED_RESCHED:	0x80000000
```

然后我用一张简易图来展示一下上面的定义：

```
    |<    8bits     >|<    8bits     >|<    8bits     >|<     8bits    >|
    +-+--------------+-----+-+--------+--------------+-+----------------+
    | |              |     | |hard irq| softirq cnt  | | preempt cnt    |
    +-+--------------+-----+-+--------+--------------+-+----------------+
     ^                      ^                         ^
     |                      |                         |
     |                      |                         |
     |                      |                         |
     |                      |                         +--- Bit8:  in_serving_softirq()
     |                      +----------------------------- Bit20: NMI_MASK
     +---------------------------------------------------- Bit31: PREEMPT_NEED_RESCHED
```

与此同时，就要引出几个和当前cpu运行状态相关的函数了。

* in_irq()       - We're in (hard) IRQ context
* in_softirq()   - We have BH disabled, or are processing softirqs
* in_interrupt() - We're in NMI,IRQ,SoftIRQ context or have BH disabled
* in_serving_softirq() - We're in softirq context
* in_nmi()       - We're in NMI context
* in_task()	     - We're in task context

原来我们通常判断cpu状态的函数，就是根据cpu上变量__preempt_count来确定的。顿时有种找到根的感觉。

那为什么会插播这么一个变量呢？是因为在softirq中，以及其他很多地方都会判断这个值来确认当前cpu运行状态，以此来判断是否应该执行什么操作。

好了，这个“软中断”结束了，让我们回到上文。

# 何时调用软中断？

对软中断的调用，还得分成两步：

  * 标记有软中断的请求
  * 在适当的时机执行

毕竟软中断不像硬中断，可以来了就执行。软中断没有硬件的这种权利打断别人的运行，只好先标记好自己的到来，等待时机的出现。

## 标记软中断请求

标记请求由函数raise_softirq(nr)来完成。

```
    raise_softirq(nr), explicit raise
        local_irq_save(flags);
        raise_softirq_irqoff(nr);
            __raise_softirq_irqoff(nr);
                or_softirq_pending(1UL << nr)
                    __this_cpu_or(local_softirq_pending_ref, (x))
            wakeup_softirqd(), if !in_interrupt()
                tsk = __this_cpu_read(ksoftirqd);
                wake_up_process(tsk); wakeup ksoftirqd
        local_irq_restore(flags);
```

实际上做了什么呢？打开看一看。

  * 关中断
  * 在变量local_softirq_pending_ref上，标记nr
  * 如果不在in_interrupt()，唤醒softirq线程

在这里我们着重要讲的是第二部，标记变量local_softirq_pending_ref。

## 在适当的时机执行软中断

之前我们也提到，软中断不像硬中断能够要求硬件及时响应。所以只好等到某个时间，再由cpu来处理。那么都有哪些时机，cpu回来处理软中断呢？

  * irq_exit():        中断处理函数返回时
  * local_bh_enable(): 允许软中断时
  * raise_softirq():   标记软中断时（通过唤醒线程）

在没有线程化irq时，前两者是立即执行的，只有第三者是通过wakeup来唤醒软中断线程。

这里我们结合一下上一小节硬插进来的概念，看看raise_softirq()中的处理。

```
    if(!in_interrupt())
        wakeup_softirq()
```

而这个in_interrupt()的含义是： We're in NMI,IRQ,SoftIRQ context or have BH disabled。这就是当我们cpu运行在这几个状态下时，我们就不去唤醒软中断了。
为什么呢？因为当我们从中断/软中断返回时，我们会去处理新的这个软中断的。

这就是上一小节引入变量的作用。

# 响应软中断

终于是时候来看软中断的响应流程了。在上面三个响应软中的地方看下来，最终都会走到函数__do_softirq()。

这个函数就是根据变量local_softirq_pending_ref上标记的软中断号，来依次处理事先注册好的软中断函数。

当然里面有几个点值得关注：

  * 函数__local_bh_disable_ip(_RET_IP_, SOFTIRQ_OFFSET)和__local_bh_enable(SOFTIRQ_OFFSET) 来表示in_serving_softirq()。
  * 保存好现场后才开中断

好了，大致框架梳理完了。有机会再来细扣其中的细节。
