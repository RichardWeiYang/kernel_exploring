这章我们先来看看如何使用ftrace。

说来惭愧，多少年前同事告诉我这种高级方法的时候，我还觉得麻烦，依然在使用printk。现在回想起来真实井底之蛙啊。

有几个非常不错的参考资料：

  * 内核源码中的[使用手册][1]
  * [LWN 1][2]
  * [LWN 2][3]
  * [Secrets of the Ftrace function tracer][6]
  * [LWN Artical List][4]
  * [Ftrace Kernel Hooks: More than just tracing][5]

接下来就从来看几个例子：

# Tracing测试文件目录

要和trace打交道都是要通过它的debugfs接口，默认情况下这个文件目录在 /sys/kernel/debug/tracing.

你也可以手动挂载到想要的位置：

```
mount -t tracefs nodev /sys/kernel/tracing
```

在这个目录下有很多重要的文件，具体每个文件的作用可以在[使用手册][1]中找到。这里解说几个比较重要的

设置当前使用的tracer

  * current_tracer
  * available_tracer

是否过滤函数

  * available_filter_functions
  * set_ftrace_filter
  * set_graph_function

得到trace输出

  * trace
  * per_cpu/cpu0/trace
  * trace_pipe

控制是否将跟踪输出写入

  * tracing_on

像我这样的初级小白，了解这么几个文件就够用了。

# 使用function tracer

先来看一个完整的使用方法。

```
echo 0 > tracing_on
echo function > current_tracer
echo 1 > tracing_on
# wait a while
echo 0 > tracing_on
cat trace
cat per_cpu/cpu0/trace
```

tracing_on用来控制是否实际输出到trace文件，所以我们在开始和结束时，我们都关掉输出，以免有太多的输出内容。
第二个命令用来指定这次使用的是function trace。因为没有设置函数过滤，所以应该是所有的内核函数都会被记录。
打开输出，停一会儿再关掉后，就可以通过trace文件查看这段时间内的内核运行情况了。

另外还可以通过查看per_cpu/cpu0/trace文件来只观察在cpu0上发生的内核函数调用。这是一个非常不错去除多个cpu之间干扰的好方法。

# 使用function_graph tracer

这个例子和之前的区别在于选择了不同的tracer，function_graph的话可以打印出函数调用的关系，更加方便理解。

```
echo 0 > tracing_on
echo function_graph > current_tracer
echo 1 > tracing_on
# wait a while
echo 0 > tracing_on
cat trace
cat per_cpu/cpu0/trace
```

也就不做过多的解释。

# 使用trace_printk()

这个函数的用法和printk()一样，只是输出不在console，而是在ftrace的ring buffer。

而且在nop tracer的情况下也能在trace中看到输出。这样就不会被函数调用信息干扰到了。

汗颜，现在才知道这个。

# 使用function command

这个用法来自作者的第三篇文章[Secrets of the Ftrace function tracer][6]。因为trace的内容太多，这个方法能够在某个函数被调用时停止trace，方便调试时观察到trace信息。

方法如下

```
# echo '__bad_area_nosemaphore:traceoff' > set_ftrace_filter
# cat set_ftrace_filter
#### all functions enabled ####
__bad_area_nosemaphore:traceoff:unlimited
```

这样就可以在函数__bad_area_nosemaphore触发时，停止trace，留住最后的案发现场。

# 现实谁调用了某个函数

在学习内核的过程中，我们有时候会想了解一下一个函数都会被谁调用，整个调用栈是什么样子的。ftrace也提供了这个功能。

```
   [tracing]# echo kfree > set_ftrace_filter
   [tracing]# cat set_ftrace_filter
   kfree
   [tracing]# echo function > current_tracer
   [tracing]# echo 1 > options/func_stack_trace
   [tracing]# cat trace | tail -8
    => sys32_execve
    => ia32_ptregs_common
                cat-6829  [000] 1867248.965100: kfree <-free_bprm
                cat-6829  [000] 1867248.965100: <stack trace>

    => free_bprm
    => compat_do_execve
    => sys32_execve
    => ia32_ptregs_common
   [tracing]# echo 0 > options/func_stack_trace
   [tracing]# echo > set_ftrace_filter
```

# Profiling

Ftrace也能做profile。

```
   [tracing]# echo nop > current_tracer
   [tracing]# echo 1 > function_profile_enabled
   [tracing]# cat trace_stat/function0 |head
     Function                               Hit    Time            Avg
     --------                               ---    ----            ---
     schedule                             22943    1994458706 us     86931.03 us
     poll_schedule_timeout                 8683    1429165515 us     164593.5 us
     schedule_hrtimeout_range              8638    1429155793 us     165449.8 us
     sys_poll                             12366    875206110 us     70775.19 us
     do_sys_poll                          12367    875136511 us     70763.84 us
     compat_sys_select                     3395    527531945 us     155384.9 us
     compat_core_sys_select                3395    527503300 us     155376.5 us
     do_select                             3395    527477553 us     155368.9 us
```

没实验成功，以后再看看。

[1]: https://github.com/torvalds/linux/blob/master/Documentation/trace/ftrace.rst
[2]: https://lwn.net/Articles/365835/
[3]: https://lwn.net/Articles/366796/
[4]: https://lwn.net/Kernel/Index/#Tracing
[5]: https://blog.linuxplumbersconf.org/2014/ocw/system/presentations/1773/original/ftrace-kernel-hooks-2014.pdf
[6]: https://lwn.net/Articles/370423/
