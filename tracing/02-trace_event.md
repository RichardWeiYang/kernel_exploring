TraceEvent是内核中一种探测的机制，据说在不使能的时候是没有损耗的。据说使用起来挺简单，但是要看懂着实需要花些力气。

# 例子

从例子中学习，一般都是比较好的方法。内核开发者也比较nice，在内核源码samples/trace_events目录下就有这么一个例子。

其中文件一共有三个：

```
total 56
-rw-r--r--  1 weiyang  staff    599 Feb 25 22:28 Makefile
-rw-r--r--  1 weiyang  staff   2936 Feb 25 22:28 trace-events-sample.c
-rw-r--r--  1 weiyang  staff  17495 May  2 14:43 trace-events-sample.h
```

这个例子以内核模块的形式存在，所以只要执行make就可以编译完成。

总的来说，要定义和使用tracepoint，只要做两点。

  * 用TRACE_EVENT来定义一个新的tracepoint
  * 在需要的地方，使用函数trace_XXX打印输出

有了例子我们就要跑一跑，来看看如何使用的。

首先我们要编译出我们的例子，这时候需要加上打开两个编译配置

  * CONFIG_SAMPLES
  * CONFIG_SAMPLE_TRACE_EVENTS

编译

  > make M=samples/trace_events

然后加载这个例子模块

  > modprobe trace-events-sample

因为用户接口在debugfs上，所以还要确保debugfs挂载了。

  > mount -t debugfs none /sys/kernel/debug/

此时我们就能在 /sys/kernel/debug/tracing/events/sample-trace/ 目录下看到该模块创建好的trace event了。

接下来，我们就可以打开这个探测时间，并且查看探测的输出了。

  > cd /sys/kernel/debug/tracing

  > echo 1 > events/sample-trace/enable

  > cat trace

  > echo 0 > events/sample-trace/enable

通过cat trace观察，可以看出系统运行时的一些状态。

让我们进一步再来看看events/sample-trace这个目录：

```
ll events/sample-trace/
total 0
-rw-r--r-- 1 root root 0 May  4 23:24 enable
-rw-r--r-- 1 root root 0 May  4 23:24 filter
drwxr-xr-x 2 root root 0 May  4 23:24 foo_bar
drwxr-xr-x 2 root root 0 May  4 23:24 foo_bar_with_cond
drwxr-xr-x 2 root root 0 May  4 23:24 foo_bar_with_fn
drwxr-xr-x 2 root root 0 May  4 23:24 foo_with_template_cond
drwxr-xr-x 2 root root 0 May  4 23:24 foo_with_template_fn
drwxr-xr-x 2 root root 0 May  4 23:24 foo_with_template_print
drwxr-xr-x 2 root root 0 May  4 23:24 foo_with_template_simple
```

可以看到

  * 目录名称sample-trace由TRACE_SYSTEM这个宏定义，所以通过查找这个宏，就能知道由多少events的大类
  * 每一个TRACE_EVENT都有一个自己的目录

源文件中trace_XXX的函数就是执行探测记录的地方了。那么这些函数是怎么定义的呢？

# TRACE_EVENT定义

看完了例子，我们就该看代码实现了。讲真，这是我见过的最长的宏展开了。之前在qemu上看到的那个hmp-command和这个比起来简直就是个小屁孩。


先来看一下例子中是如何定义一个trace event的。和其他定义不同，定义trace event的定义在头文件，而非源文件。
我把trace-events-sample.h文件做一个简要的打开。

```
#include <linux/tracepoint.h>

TRACE_EVENT(foo_bar, ...)

...

#undef TRACE_INCLUDE_PATH
#undef TRACE_INCLUDE_FILE
#define TRACE_INCLUDE_PATH .

#define TRACE_INCLUDE_FILE trace-events-sample
#include <trace/define_trace.h>
```

中间我省略了很多TRACE_EVENT及其变体，每一个TRACE_EVENT对应了一个trace point。

可以看到，一个trace event的定义需要涉及到起码两个头文件。

## 史上最长宏定义

你以为就这么简单吗？当然不是，作为有多年阅读c语言代码的老司机，看到真正的定义，我都差点没有吐出来。。。

好了，不扯淡了。怎么能很好的解释这个宏展开的过程呢？还是用一张图吧。倒吸一口气，准备一次无尽的代码阅读。

```
        trace-events-sample.h
        +----------------------------------------------------------------+
        |#define TRACE_SYSTEM      sample-trace                          |
        |#define TRACE_SYSTEM_VAR  sample_trace                          |
        |                                                                |
        |#defined(TRACE_HEADER_MULTI_READ)                               |
        |#include <linux/tracepoint.h>                                   |
        |                                                                |
        |   #define TRACE_EVENT(name) DECLARE_TRACE(name)                |
        |   #define DECLARE_TRACE(name) __DECLARE_TRACE(name)            |
        |   #define __DECLARE_TRACE()                                   \|
        |           struct tracepoint __tracepoint_##name;              \|
        |           void trace_##name() {}                              \|
        |           int register_trace_##name() {                       \|
        |               tracepoint_probe_register();                    \|
        |           }                                                   \|
        |           int register_trace_prio_##name() {                  \|
        |               tracepoint_probe_register_prio();               \|
        |           }                                                   \|
        |           int unregister_trace_##name() {                     \|
        |               tracepoint_probe_unregister();                  \|
        |           }                                                   \|
        |           void check_trace_callback_type_##name()             \|
        |           bool trace_##name_enabled()                         \|
        |                                                                |
        |TRACE_EVENT(foo_bar, );                                         |  <- expand 1st time
        |#endif  // TRACE_HEADER_MULTI_READ                              |
        |                                                                |
        |#define TRACE_INCLUDE_FILE trace-events-sample                  |
        |#include <trace/define_trace.h>                                 |
        |                                                                |
        |   #undef TRACE_EVENT()                                         |
        |   #define TRACE_EVENT() DEFINE_TRACE()                         |
        |   #define DEFINE_TRACE() DEFINE_TRACE_FN()                    \|  # in "linux/tracepoint.h"
        |           char __tpstrtab_##name[] = #name;                   \|
        |                section("__tracepoints_strings")               \|
        |           struct tracepoint __tracepoint_##name =             \|
        |              {                                                \|
        |               __tpstrtab_##name,                              \|
        |               STATIC_KEY_INIT_FALSE,                          \|
        |               reg,      = NULL                                \|
        |               unreg,    = NULL                                \|
        |               NULL,                                           \|
        |              };                                               \|
        |                section("__tracepoints")                       \|
        |           __TRACEPOINT_ENTRY(name);                            |
        |   #define __TRACEPOINT_ENTRY(name)                            \|
        |           tracepoint_ptr_t __tracepoint_ptr_##name            \|
        |           __attribute__((section("__tracepoints_ptrs"))) =    \|
        |                 &__tracepoint_##name                           |
        |                                                                |
        |                                                                |
        |   #define TRACE_HEADER_MULTI_READ                              |
        |   #include "trace-events-sample.h"                             |  # due to TRACE_HEADER_MULTI_READ
        |                                                                |  # we just have TRACE_EVENT() left
        |      TRACE_EVENT()                                             |  <- expand 2nd time
        |                                                                |
        |                                                                |
        |   #include <trace/trace_events.h>                              |
        |                                                                |
        |      #undef TRACE_EVENT()                                      |
        |      #define TRACE_EVENT()                                    \|
        |              DECLARE_EVENT_CLASS()                            \|
        |              DEFINE_EVENT()                                    |
        |      #undef DECLARE_EVENT_CLASS()                              |
        |      #define DECLARE_EVENT_CLASS()                            \|
        |              struct trace_event_raw_##name {};                \|
        |              struct trace_event_class event_class_##name;      |
        |      #undef DEFINE_EVENT()                                     |
        |      #define DEFINE_EVENT()                                    |
        |              struct trace_event_call event_##name;             |
        |                                                                |
        |      #include "trace-events-sample.h"                          |
        |         TRACE_EVENT()                                          |  <- expand 3rd time
        |                                                                |
        |      #undef DECLARE_EVENT_CLASS()                              |
        |      #define DECLARE_EVENT_CLASS()                            \|
        |              struct trace_event_data_offset_##name {}          |
        |                                                                |
        |      #include "trace-events-sample.h"                          |
        |         TRACE_EVENT()                                          |  <- expand 4th time
        |                                                                |
        |      #undef DECLARE_EVENT_CLASS()                              |
        |      #define DECLARE_EVENT_CLASS( , , , , , print)            \|
        |              enum print_line_t trace_raw_output_##name {      \|
        |                     trace_seq_printf(s, print);               \|
        |              }                                                \|
        |              struct trace_event_functions                     \|
        |                       trace_event_type_funcs_##name {         \|
        |                       .trace = trace_raw_output_##name,       \|
        |                     }                                          |
        |                                                                |
        |      #include "trace-events-sample.h"                          |
        |         TRACE_EVENT()                                          |  <- expand 5th time
        |                                                                |
        |      #undef DECLARE_EVENT_CLASS()                              |
        |      #define DECLARE_EVENT_CLASS()                            \|
        |              struct trace_event_fields                        \|
        |                     trace_event_fields_##name[] = { tstruct }; |
        |                                                                |
        |      #include "trace-events-sample.h"                          |
        |         TRACE_EVENT()                                          |  <- expand 6th time
        |                                                                |
        |      #undef DECLARE_EVENT_CLASS()                              |
        |      #define DECLARE_EVENT_CLASS()                            \|
        |              int trace_event_get_offsets_##name(              \|
        |                  struct trace_event_data_offset_##name off,   \|
        |                  proto) {}                                     |
        |                                                                |
        |      #include "trace-events-sample.h"                          |
        |         TRACE_EVENT()                                          |  <- expand 7th time
        |                                                                |
        |      #undef DECLARE_EVENT_CLASS()                              |
        |      #define DECLARE_EVENT_CLASS()                            \|
        |              void trace_event_raw_event_##name() {}            |
        |      #undef DEFINE_EVENT()                                     |
        |      #define DEFINE_EVENT()                                   \|
        |              void ftrace_test_probe_##name() {}                |
        |                                                                |
        |      #include "trace-events-sample.h"                          |
        |         TRACE_EVENT()                                          |  <- expand 8th time
        |                                                                |
        |      #undef DECLARE_EVENT_CLASS()                              |
        |      #define DECLARE_EVENT_CLASS()                            \|
        |              char print_fmt_##name[] = print;                 \|
        |              struct trace_event_class event_class_##name = {}; |
        |      #undef DEFINE_EVENT()                                     |
        |      #define DEFINE_EVENT()                                   \|
        |              struct trace_event_call event_##name = {};       \|
        |              struct trace_event_call *__event_##name =        \|
        |                     section("_ftrace_events") &event_##name;   |
        |                                                                |
        |      #include "trace-events-sample.h"                          |
        |         TRACE_EVENT()                                          |  <- expand 9th time
        |                                                                |
        +----------------------------------------------------------------+
```

终于完了，也不知道有没有漏掉什么。。。大家如果真的想要看实际代码中展开后的代码，可以运行

> make samples/trace_events/trace-events-sample.i

生成的文件是经过预处理后得到的源代码。不过相信我，你可能不太会愿意去看这个（捂脸）

回过头来再看这展开，让我们来总结一下这个过程：

  * 一共包含了两个头文件：linux/tracepoint.h 和 trace/define_trace.h
  * 在trace/define_trace.h中，反复定义了TRACE_EVENT且再次包含samples/trace_events/trace-events-sample.h，实现了一个宏定义多次展开的效果


## 究竟定义了什么？

哪怕有了上面这个图，我想大部分人也是不会去看的。或者说，看了可能也不知道这些宏展开究竟定义了些什么？

> 帮人帮到底，送佛送到西

既然都帮大家做了宏展开，那我就干脆再用一张图展示一下这么多宏定义究竟定义了些什么。

```
    trace_event_call
    +------------------------------+
    |list                          |
    |    (struct list_head)        |
    |name | tp                     |  = &__tracepoint_##name
    |(char * | struct tracepoint *)|
    |    +-------------------------+
    |    |name                     |  = "name"
    |    |    (char *)             |
    |    |key                      |  = STATIC_KEY_INIT_FALSE
    |    |    (struct static_key)  |
    |    |regfunc                  |  = NULL
    |    |unregfunc                |  = NULL
    |    |    ()                   |
    |    |funcs                    |  = NULL
    |    |(struct tracepoint_func*)|    first entry is assigned to
    |    |                         |    trace_event_raw_event_##name
    |    |                         |    by tracepoint_probe_register
    |    |                         |
    |    +-------------------------+
    |class                         |  = &event_class_##name
    |   (struct trace_event_class*)|
    |    +-------------------------+
    |    |name                     |  = TRACE_SYSTEM_STRING
    |    |    (char *)             |
    |    |probe                    |  = trace_event_raw_event_##name
    |    |    (void *)             |
    |    |reg                      |  = trace_event_reg
    |    |    (void *)             |
    |    |raw_init                 |  = trace_event_raw_init
    |    |    (int *)              |
    |    |fields_array             |  = trace_event_fields_##name
    |    |   (trace_event_fields *)|
    |    +-------------------------+
    |event                         |  # be registered by register_trace_event()
    |    (struct trace_event)      |
    |    +-------------------------+
    |    |funcs                    |  = &trace_event_type_funcs_##name
    |    |  (trace_event_functions)|
    |    |  +----------------------+
    |    |  |trace                 |  = trace_raw_output_##name -> trace_seq_printf(s, print)
    |    |  |raw                   |  = trace_nop_print
    |    |  |hex                   |  = trace_nop_print
    |    |  |binary                |  = trace_nop_print
    |    |  |    (trace_print_func)|
    |    +--+----------------------+
    |flags                         |
    |    (int)                     |
    |print_fmt                     |  = print_fmt_##name
    |    (char *)                  |
    |                              |
    +------------------------------+
```

经过了一番云里雾里的宏展开，实际上就是(主要)定义出了这么一个数据结构 -- **trace_event_call**。而且这个数据结构关联了几个重要的小伙伴

  * tracepoint
  * trace_event_class
  * trace_event

后续，我们将逐渐看到在初始化和使能的过程中，这些数据结构之间的爱恨情仇。

# 注册trace_event

有了数据结构，想要使用这个功能，我们能想到的第一步就是要把相关的数据结构注册到某个地方，这样下次才能够被使用到是不是？

这个秘密隐藏在了刚才宏展开的最后一次展开中，大家可以回过去搜“section("_ftrace_events") &event_##name;”。有内核代码经验的朋友可能已经才到了，这个意思是我们把一部分的内容强制保存在了一个名为_ftrace_events的section中。这些是什么呢？对了就是trace_event_call结构体的指针们。

有了这个信息，我们再来看链接文件的定义：

```
#define FTRACE_EVENTS()	. = ALIGN(8);					\
			__start_ftrace_events = .;			\
			KEEP(*(_ftrace_events))				\
			__stop_ftrace_events = .;			\
			__start_ftrace_eval_maps = .;			\
			KEEP(*(_ftrace_eval_map))			\
			__stop_ftrace_eval_maps = .;
```

我们看到_ftrace_events这个section被包含在__start_ftrace_events之间。那就沿着这条线继续。

```
struct trace_event_call **iter, *call;

for_each_event(iter, __start_ftrace_events, __stop_ftrace_events) {

  call = *iter;
  ret = event_init(call);
      call->class->raw_init(call);
  if (!ret)
    list_add(&call->list, &ftrace_events);
}
```

看到了么？ 我们依次从__start|stop_ftrace_events之间拿出每一个内容，再执行event_init()。而这个类型正好是trace_event_call，和刚才的定义吻合上。

但是event_init()里面又调用了什么call->class->raw_init(call)，这是什么个鬼？别急，这个我已经给你写好了。请跳回到刚才解释trace_event_call的图上找找，这个raw_init函数就是trace_event_raw_init。最后这个通过register_trace_event将trace_event_call.event注册到系统中，而这个event的类型是trace_event。

怎么样，是不是够刺激的？

最后我们再来展示一下trace_event注册到系统中后的样子吧。

```
   ftrace_event_list(struct list_head)
     |
     |  struct trace_event
     +->+------------------------------+
        |list                          |
        |    (struct list_head)        |
        |node                          |
        |    (struct hlist_node)       |
        |type                          |  = sort of index in ftrace_event_list
        |    (int)                     |    also a key in event_hash[]
        |funcs                         |  = &trace_event_type_funcs_##name
        |  (trace_event_functions)     |
        |   +--------------------------+
        |   |trace                     |  = trace_raw_output_##name -> trace_seq_printf(s, print)
        |   |raw                       |  = trace_nop_print
        |   |hex                       |  = trace_nop_print
        |   |binary                    |  = trace_nop_print
        |   |    (trace_print_func)    |
        +---+--------------------------+

   event_hash[128] (sturct hlist_head)
```

trace_event结构会在两个地方注册：

  * ftrace_event_list： 这个链表用来遍历事件的号码
  * event_hash[128]:    这个哈希表用来查找

有没有看到其中funcs的成员第一个是之前定义的 trace_raw_output_##name？我猜这个就是最后输出到trace文件的代码，你觉得呢？

好了，数据结构注册完了，接下来是什么呢？

# 注册trace_event_call

在上一节中，我们看到了内核通过编译链接的方法找到了trace_event_call，并且将其中的trace_event注册到了系统中，现在我们来看看trace_event_call是如何注册到系统中的。

这个过程就在event_init()函数下面一点。一共有两个步骤：

  * 添加到ftrace_events链表
  * 添加到trace_array的events

第一步就在刚才的代码片段中list_add(&call->list, &ftrace_events)，而第二步则是通过函数__trace_early_add_events()。

```
__trace_early_add_events()
    list_for_each(call, &ftrace_events, list)
    __trace_early_add_new_event(call, tr)
```

经过这次注册，将trace_event_call和trace_array连接了起来：

```
      struct trace_array
      +--------------------------------+
      |events                          |
      |    (struct list_head)          |
      |       ^                        |
      |       |                        |
      +--------------------------------+
              |                         
              |
              |
              |      trace_event_file                trace_event_file                trace_event_file
              |      +--------------------------+    +--------------------------+    +--------------------------+
              +----->|list                      |<-->|list                      |<-->|list                      |
                     |     (struct list)        |    |     (struct list)        |    |     (struct list)        |
                     |event_call                |    |event_call                |    |event_call                |
                     |     (trace_event_call *) |    |     (trace_event_call *) |    |     (trace_event_call *) |
                     |                          |    |                          |    |                          |
                     +--------------------------+    +--------------------------+    +--------------------------+
```


# 创建tracefs


在使用trace工具的时候，会通过tracefs往某些文件里读写来控制ftrace。trace_event也不例外，所以我们要先来看一下tracefs的构建，为后续的代码阅读做好准备。

说起来这个过程有点绕，因为创建tracefs的地方和刚才那些注册函数不在一个地方（系统启动时）。

```
    event_trace_init()
        tr = top_trace_array()
        d_tracer = tracing_init_dentry()
        tracefs_create_file("available_events", ,tr, &ftrace_avail_fops)
        trace_define_generic_fields()
        trace_define_common_fields()
        early_event_add_tracer(d_tracer, tr)
            create_event_toplevel_files(d_tracer, tr)
                /sys/kernel/debug/tracing/set_event, ftrace_set_event_fops
                /sys/kernel/debug/tracing/events
                /sys/kernel/debug/tracing/events/enable, ftrace_tr_enable_fops
                /sys/kernel/debug/tracing/set_event_pid, ftrace_set_event_pid_fops
                /sys/kernel/debug/tracing/set_event_notrace_pid, ftrace_set_event_notrace_pid_fops
                /sys/kernel/debug/tracing/events/header_page, ftrace_show_header_fops
                /sys/kernel/debug/tracing/events/header_event, ftrace_show_header_fops
                tr->event_dir = /sys/kernel/debug/tracing/events/
            __trace_early_add_event_dirs(tr)
                event_create_dir(tr->event_dir, file), iterate on tr->events
        register_module_notifier(&trace_module_nb)
```

具体细节可以看源代码，这里解释两点：

  * create_event_toplevel_files 创建了和trace event相关的根目录的一些文件
  * event_create_dir则是会对每一个trace_array->events上的trace_event_file调用，创建每个event的目录

而这个trace_array->events则是由, 刚才看到的函数__trace_early_add_new_event()添加的。

# 初始化过程的梳理

到这里估计你已经晕了，没事我自己写得也晕了。让我们来梳理一下整个初始化过程，明确一下这个注册和tracefs的创建顺序。


```
start_kernel
    trace_init
        event_trace_enable
            event_init(trace_event_call)                        --- (1)
            list_add(trace_event_call, &ftrace_events)          --- (2)
        __trace_early_add_event()
            list_for_each_entry(call, &ftrace_events, list)
            __trace_early_add_new_event()
                trace_create_new_event()                        --- (3)

tracer_init_tracefs, fs_initcall()
    event_trace_init()
        early_event_add_tracer()
            __trace_early_add_event_dirs()                      --- (4)
```

  * (1) 从特定的section中拿到trace_event_call数据结构，并注册了trace_event
  * (2) 将trace_event_call添加到了ftrace_events链表
  * (3) 将每一个trace_event_call以trace_event_file的形式添加到trace_array.events
  * (4) 为每一个trace_array.events创建自己的tracefs

废了这么大力气我们都做了什么呢？

> 关联了tracefs和trace_event_file，也就是我嗯定义的trace_event_call。

所以，每当我们操作一个tracefs文件的时候，后面就对应这相应的trace_event_file和trace_event_call了。

```
    /sys/kernel/debug/tracing/events/sample-events/enable
        |
        |
        |
        v
    trace_event_file          trace_event_call
    +-------------------+     +-------------------+
    |event_call         |---->|                   |
    |                   |     |                   |
    +-------------------+     +-------------------+
```

OK, 我们已经为tracefs的操作做好了准备，让我们来看看打开trace event选项时的动作吧。

# 打开事件

在查看trace文件中的事件记录前，我们需要使能这个事件。

>  echo 1 > events/sample-trace/enable

所以有个开关来控制事件。而当我们写这个文件的时候，触发到的内核函数就是刚才我们注册tracefs对应的ops中的event_enable_write。

```
event_enable_write()
    ftrace_event_enable_disable(file, val) -> __ftrace_event_enable_disable()
        call->class->reg() -> trace_event_reg()
            tracepoint_probe_register(call->tp, class->class->probe, file)
                tracepoint_add_func(tp, &tp_func, prio)
```

绕晕了，其实呢就是通过某种方式设置了tracepoint结构体中的funcs成员。刚才我们在trace_event_call结构体中已经看到了tracepoint结构，这次该好好看一眼了。

```
     tracepoint
     +------------------------------+
     |name                          |  = "name"
     |    (char *)                  |
     |key                           |  = STATIC_KEY_INIT_FALSE
     |    (struct static_key)       |
     |regfunc                       |  = NULL
     |unregfunc                     |  = NULL
     |    ()                        |
     |funcs                         |
     |    (struct tracepoint_func*) |
     |    +-------------------------+
     |    |func                     |  = trace_event_raw_event_##name
     |    |data                     |  = &trace_event_file
     |    |    (void *)             |
     |    |prio                     |
     |    |    (int)                |
     |    +-------------------------+
     |    |func                     |  = NULL  
     |    |data                     |
     |    |    (void *)             |
     |    |prio                     |
     |    |    (int)                |
     +----+-------------------------+
```

主角终于登场了，经过这么一顿骚操作后，我们将之前定义好的 trace_event_raw_event_##name挂到了tracepoint的funcs列表中。当然我还省去了重要的一步--设置key。

# 输出事件

终于到了最后了。之前说的都是定义和初始化，终于要看到调用的情况了。在例子中我们看到，当我们需要输出一个事件时，就会调用trace_XXX()。这次该轮到它出场了。

先来看看trace_XXX这个函数的定义，它也藏在了我们刚才宏定义的展开中，这次我们仔细看一眼

```
	static inline void trace_##name(proto)				      \
	{								                                    \
		if (static_key_false(&__tracepoint_##name.key))		\
			__DO_TRACE(&__tracepoint_##name,		            \
				TP_PROTO(data_proto),			                    \
				TP_ARGS(data_args),			                      \
				TP_CONDITION(cond), 0);			                  \
		if (IS_ENABLED(CONFIG_LOCKDEP) && (cond)) {		    \
			rcu_read_lock_sched_notrace();			            \
			rcu_dereference_sched(__tracepoint_##name.funcs);\
			rcu_read_unlock_sched_notrace();		            \
		}							                                    \
	}								                                    \
```

每次我们调用trace_XXX()函数的时候，先检查key是否使能了，如果使能了才继续往下走。接着我们再打开__DO_TRACE来看看。

```
#define __DO_TRACE(tp, proto, args, cond, rcuidle)			\
	do {								                                  \
		struct tracepoint_func *it_func_ptr;			          \
		void *it_func;						                          \
		void *__data;						                            \
									                                      \
		it_func_ptr = rcu_dereference_raw((tp)->funcs);		  \
									                                      \
		if (it_func_ptr) {					                        \
			do {						                                  \
				it_func = (it_func_ptr)->func;		              \
				__data = (it_func_ptr)->data;		                \
				((void(*)(proto))(it_func))(args);	            \
			} while ((++it_func_ptr)->func);		              \
		}							                                      \
									                                      \
		preempt_enable_notrace();				                    \
	} while (0)
```

联系上上一小节的tracepoint结构体是不是能想到啥？对了，就是遍历tracepoint->funcs数组，然后调用它们。

好了，终于完整得看完了TRACE_EVENT的定义和使用流程。小编累了，大家也累了，今天就到这里吧。
