随着对内核理解的深入，是时候了解一下对内核调试、跟踪、画像的技术了。

不仅可以帮助自己了解内核运行，更能在出现性能问题时定位瓶颈。

在内核中能帮助我们的工具有很多，让我一一学来。

经过一段时间学习，发现内核中很多探测方法都基于了ftrace提供的框架。比如在[ftrace 中 eventtracing 的实现原理][3]中提到的trace原理，以及其他探测机制是如何使用ftrace框架的。在这里也把文中的图放在这里，做一个直观的了解。

![ftrace framework](/tracing/ftrace_framework.png)

既然如此，那我们就先来看看ftrace的使用以及原理。

[ftrace的使用][4]

[探秘ftrace][5]

接着来看的是传说为瑞士军刀的[eBPF初探][1].
严重怀疑后续可能要完整的一章来讲述清楚。

对eBPF的探索中，发现他和内核系统中其他的探测机制有着非常深厚的关联。

比如:

[TraceEvent][2]

[1]: /tracing/01-ebpf.md
[2]: /tracing/02-trace_event.md
[3]: https://www.ibm.com/developerworks/cn/linux/1609_houp_ftrace/index.html
[4]: /tracing/03-ftrace_usage.md
[5]: /tracing/04-ftrace_internal.md
