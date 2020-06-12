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

总的来说，要定义和使用tracepoint，只要坐两点。

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

# TRACE_EVENT的定义

和其他组件不同，定义trace event的复杂点不是在源文件，而是在头文件中。

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

## 第一次重复包含

原本来讲TRACE_EVENT一看就是一个宏定义，找到它并且展开就能清楚trace point是如何定义的了。
但是这个宏却偏偏有点出角，而神奇的地方就在trace/define_trace.h和相关的头文件中。

那我先把trace/define_trace.h做一个简单的展开。

```
#undef TRACE_EVENT
#define TRACE_EVENT(name, proto, args, tstruct, assign, print)	\
	DEFINE_TRACE(name)

...

#ifndef TRACE_INCLUDE_PATH
# define __TRACE_INCLUDE(system) <trace/events/system.h>
# define UNDEF_TRACE_INCLUDE_PATH
#else
# define __TRACE_INCLUDE(system) __stringify(TRACE_INCLUDE_PATH/system.h)
#endif

# define TRACE_INCLUDE(system) __TRACE_INCLUDE(system)

/* Let the trace headers be reread */
#define TRACE_HEADER_MULTI_READ

#include TRACE_INCLUDE(TRACE_INCLUDE_FILE)                  ---  (1)

#include <trace/trace_events.h>
#include <trace/perf.h>
#include <trace/bpf_probe.h>

#undef TRACE_HEADER_MULTI_READ
```

这块有点绕，因为在trace-events-sample.h中定义了TRACE_INCLUDE_PATH，所以导致了在上面代码的(1)处，再次包含了trace-events-sample.h。
而前后的TRACE_HEADER_MULTI_READ操作，保证了trace-events-sample.h能够被第二次包含。


## 第二次重复包含

但是你一个就这么重复包含一次就算完了么？早着呢。

神奇的地方就在trace/trace_events.h中了。让我再一次来展开吧：

```
#include <linux/trace_events.h>

#undef TRACE_EVENT
#define TRACE_EVENT(name, proto, args, tstruct, assign, print) \
	DECLARE_EVENT_CLASS(name,			       \
			     PARAMS(proto),		       \
			     PARAMS(args),		       \
			     PARAMS(tstruct),		       \
			     PARAMS(assign),		       \
			     PARAMS(print));		       \
	DEFINE_EVENT(name, name, PARAMS(proto), PARAMS(args));

...

#include TRACE_INCLUDE(TRACE_INCLUDE_FILE)         --- (1)

#undef DEFINE_EVENT
#define DEFINE_EVENT(template, name, proto, args)

...

#include TRACE_INCLUDE(TRACE_INCLUDE_FILE)         --- (2)

...

#include TRACE_INCLUDE(TRACE_INCLUDE_FILE)         --- (3)

...

#include TRACE_INCLUDE(TRACE_INCLUDE_FILE)         --- (4)

...

#include TRACE_INCLUDE(TRACE_INCLUDE_FILE)         --- (5)

...

#include TRACE_INCLUDE(TRACE_INCLUDE_FILE)         --- (6)

...

#include TRACE_INCLUDE(TRACE_INCLUDE_FILE)         --- (7)
```

也就是说在trace/trace_events.h，先后其次包含了TRACE_INCLUDE_FILE文件，也就是这个例子中的trace-events-sample.h。

换句话说每一个TRACE_EVENT定义的过程中都要 7 + 1次展开。反正我已经是崩溃了。

## 手动展开TRACE_EVENT

### 第一次展开

好了，我们回来关注我们的重点TRACE_EVENT的定义。在define_trace.h头文件中的开始处有一个TRACE_EVENT的定义。因为在trace-events-sample.h中我们包含了一个头文件linux/tracepoint.h。所以这时TRACE_EVENT的定义在linux/tracepoint.h中：

```
#define __DECLARE_TRACE(name, proto, args, cond, data_proto, data_args) \
	extern struct tracepoint __tracepoint_##name;			\
	static inline void trace_##name(proto)				\
	{								\
		if (static_key_false(&__tracepoint_##name.key))		\
			__DO_TRACE(&__tracepoint_##name,		\
				TP_PROTO(data_proto),			\
				TP_ARGS(data_args),			\
				TP_CONDITION(cond), 0);			\
		if (IS_ENABLED(CONFIG_LOCKDEP) && (cond)) {		\
			rcu_read_lock_sched_notrace();			\
			rcu_dereference_sched(__tracepoint_##name.funcs);\
			rcu_read_unlock_sched_notrace();		\
		}							\
	}								\
	__DECLARE_TRACE_RCU(name, PARAMS(proto), PARAMS(args),		\
		PARAMS(cond), PARAMS(data_proto), PARAMS(data_args))	\
	static inline int						\
	register_trace_##name(void (*probe)(data_proto), void *data)	\
	{								\
		return tracepoint_probe_register(&__tracepoint_##name,	\
						(void *)probe, data);	\
	}								\
	static inline int						\
	register_trace_prio_##name(void (*probe)(data_proto), void *data,\
				   int prio)				\
	{								\
		return tracepoint_probe_register_prio(&__tracepoint_##name, \
					      (void *)probe, data, prio); \
	}								\
	static inline int						\
	unregister_trace_##name(void (*probe)(data_proto), void *data)	\
	{								\
		return tracepoint_probe_unregister(&__tracepoint_##name,\
						(void *)probe, data);	\
	}								\
	static inline void						\
	check_trace_callback_type_##name(void (*cb)(data_proto))	\
	{								\
	}								\
	static inline bool						\
	trace_##name##_enabled(void)					\
	{								\
		return static_key_false(&__tracepoint_##name.key);	\
	}

#define DECLARE_TRACE(name, proto, args)				\
	__DECLARE_TRACE(name, PARAMS(proto), PARAMS(args),		\
			cpu_online(raw_smp_processor_id()),		\
			PARAMS(void *__data, proto),			\
			PARAMS(__data, args))

#define TRACE_EVENT(name, proto, args, struct, assign, print)	\
  DECLARE_TRACE(name, PARAMS(proto), PARAMS(args))
```

这里我们就看到定义了

  * trace_XXX()
  * register_trace_XXX()
  * unregister_trace_XXX()

而trace_xxx()的函数，它的实现由__DO_TRACE()展开。

### 第二次展开

接下来的事情都在include/trace/trace_events.h中。

```
#undef TRACE_EVENT
#define TRACE_EVENT(name, proto, args, tstruct, assign, print) \
	DECLARE_EVENT_CLASS(name,			       \
			     PARAMS(proto),		       \
			     PARAMS(args),		       \
			     PARAMS(tstruct),		       \
			     PARAMS(assign),		       \
			     PARAMS(print));		       \
	DEFINE_EVENT(name, name, PARAMS(proto), PARAMS(args));


#undef DECLARE_EVENT_CLASS
#define DECLARE_EVENT_CLASS(name, proto, args, tstruct, assign, print)	\
	struct trace_event_raw_##name {					\
		struct trace_entry	ent;				\
		tstruct							\
		char			__data[0];			\
	};								\
									\
	static struct trace_event_class event_class_##name;

#undef DEFINE_EVENT
#define DEFINE_EVENT(template, name, proto, args)	\
	static struct trace_event_call	__used		\
	__attribute__((__aligned__(4))) event_##name
```

再次包含trace-events-sample.h前，重新定义了TRACE_EVENT和相关的定义。看上去这次定义了一个类型和两个静态变量。

  * event_class_##name
  * event_##name


### 第三次展开

接着trace/trace_events.h中又重新定义了DECLARE_EVENT_CLASS和DEFINE_EVENT。

```
#undef DECLARE_EVENT_CLASS
#define DECLARE_EVENT_CLASS(call, proto, args, tstruct, assign, print)	\
	struct trace_event_data_offsets_##call {			\
		tstruct;						\
	};

#undef DEFINE_EVENT
#define DEFINE_EVENT(template, name, proto, args)
```

这里定义了新的结构体, trace_event_data_offsets_##call。

### 第四次展开

```
#undef DECLARE_EVENT_CLASS
#define DECLARE_EVENT_CLASS(call, proto, args, tstruct, assign, print)	\
static notrace enum print_line_t					\
trace_raw_output_##call(struct trace_iterator *iter, int flags,		\
			struct trace_event *trace_event)		\
{									\
	struct trace_seq *s = &iter->seq;				\
	struct trace_seq __maybe_unused *p = &iter->tmp_seq;		\
	struct trace_event_raw_##call *field;				\
	int ret;							\
									\
	field = (typeof(field))iter->ent;				\
									\
	ret = trace_raw_output_prep(iter, trace_event);			\
	if (ret != TRACE_TYPE_HANDLED)					\
		return ret;						\
									\
	trace_seq_printf(s, print);					\
									\
	return trace_handle_return(s);					\
}									\
static struct trace_event_functions trace_event_type_funcs_##call = {	\
	.trace			= trace_raw_output_##call,		\
};
```

这里重新定义了DECLARE_EVENT_CLASS，给出一个函数 trace_raw_output_##call。

### 第五次展开

```
#undef DECLARE_EVENT_CLASS
#define DECLARE_EVENT_CLASS(call, proto, args, tstruct, func, print)	\
static struct trace_event_fields trace_event_fields_##call[] = {	\
	tstruct								\
	{} };

#undef DEFINE_EVENT
#define DEFINE_EVENT(template, name, proto, args)
```

这次定义了 trace_event_fields_##call[]。

### 第六次展开

```
#undef DECLARE_EVENT_CLASS
#define DECLARE_EVENT_CLASS(call, proto, args, tstruct, assign, print)	\
static inline notrace int trace_event_get_offsets_##call(		\
	struct trace_event_data_offsets_##call *__data_offsets, proto)	\
{									\
	int __data_size = 0;						\
	int __maybe_unused __item_length;				\
	struct trace_event_raw_##call __maybe_unused *entry;		\
									\
	tstruct;							\
									\
	return __data_size;						\
}
```

这次定义了一个函数 trace_event_get_offset_##call()。

### 第七次展开

```
#undef DECLARE_EVENT_CLASS
#define DECLARE_EVENT_CLASS(call, proto, args, tstruct, assign, print)	\
									\
static notrace void							\
trace_event_raw_event_##call(void *__data, proto)			\
{									\
	struct trace_event_file *trace_file = __data;			\
	struct trace_event_data_offsets_##call __maybe_unused __data_offsets;\
	struct trace_event_buffer fbuffer;				\
	struct trace_event_raw_##call *entry;				\
	int __data_size;						\
									\
	if (trace_trigger_soft_disabled(trace_file))			\
		return;							\
									\
	__data_size = trace_event_get_offsets_##call(&__data_offsets, args); \
									\
	entry = trace_event_buffer_reserve(&fbuffer, trace_file,	\
				 sizeof(*entry) + __data_size);		\
									\
	if (!entry)							\
		return;							\
									\
	tstruct								\
									\
	{ assign; }							\
									\
	trace_event_buffer_commit(&fbuffer);				\
}
/*
 * The ftrace_test_probe is compiled out, it is only here as a build time check
 * to make sure that if the tracepoint handling changes, the ftrace probe will
 * fail to compile unless it too is updated.
 */

#undef DEFINE_EVENT
#define DEFINE_EVENT(template, call, proto, args)			\
static inline void ftrace_test_probe_##call(void)			\
{									\
	check_trace_callback_type_##call(trace_event_raw_event_##template); \
}
```

这次有点长，展开了一个函数trace_event_raw_event_##call。

### 第八次展开

```
#undef DECLARE_EVENT_CLASS
#define DECLARE_EVENT_CLASS(call, proto, args, tstruct, assign, print)	\
_TRACE_PERF_PROTO(call, PARAMS(proto));					\
static char print_fmt_##call[] = print;					\
static struct trace_event_class __used __refdata event_class_##call = { \
	.system			= TRACE_SYSTEM_STRING,			\
	.fields_array		= trace_event_fields_##call,		\
	.fields			= LIST_HEAD_INIT(event_class_##call.fields),\
	.raw_init		= trace_event_raw_init,			\
	.probe			= trace_event_raw_event_##call,		\
	.reg			= trace_event_reg,			\
	_TRACE_PERF_INIT(call)						\
};

#undef DEFINE_EVENT
#define DEFINE_EVENT(template, call, proto, args)			\
									\
static struct trace_event_call __used event_##call = {			\
	.class			= &event_class_##template,		\
	{								\
		.tp			= &__tracepoint_##call,		\
	},								\
	.event.funcs		= &trace_event_type_funcs_##template,	\
	.print_fmt		= print_fmt_##template,			\
	.flags			= TRACE_EVENT_FL_TRACEPOINT,		\
};									\
static struct trace_event_call __used					\
__attribute__((section("_ftrace_events"))) *__event_##call = &event_##call
```

终于完了，也不知道有没有漏掉什么。。。

这次呢定义了一个变量 event_class_##call和一个定义在section("ftrace_events")的变量__event_##call。

所以接下来要发生什么呢？

# 注册tracepoint

我们看了这么多，还只是看了TRACE_EVENT的定义，但是定义了这么多东西什么时候用到还一点不知道。

还记得之前最后定义的那个_ftrace_events么？就从这里开始找吧。

```
#define FTRACE_EVENTS()	. = ALIGN(8);					\
			__start_ftrace_events = .;			\
			KEEP(*(_ftrace_events))				\
			__stop_ftrace_events = .;			\
			__start_ftrace_eval_maps = .;			\
			KEEP(*(_ftrace_eval_map))			\
			__stop_ftrace_eval_maps = .;
```

我们看到section被包含在__start_ftrace_events之间。那就沿着这条线继续。

```
struct trace_event_call **iter, *call;

for_each_event(iter, __start_ftrace_events, __stop_ftrace_events) {

  call = *iter;
  ret = event_init(call);
  if (!ret)
    list_add(&call->list, &ftrace_events);
}
```

看到了么？ 我们从__start|stop_ftrace_events之间拿出每一个内容，再执行event_init()。而这个类型正好是trace_event_call，和刚才的定义吻合上。

函数event_init()的主要工作是将 trace_event_call.event添加到哈希表event_hash上。

好了，感觉线索又断了。

#
