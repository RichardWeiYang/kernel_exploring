因为ftrace的特殊地位，打算进一步了解ftrace的一些原理。虽然只探索到一点皮毛，也希望能够记录下来为以后的工作和学习留下点笔记。

# 参考资料

先说一个非常重要的参考资料，那就是ftrace作者的pdf。没有这个pdf，我估计连门都没有摸到。

[Ftrace Kernel Hooks: More than just tracing][1]

作为一个读者，让我尝试来理解作者的思路：

  * 提供一种插入函数探针的方式
  * 为了性能（及动态使能），需要记录所有探针位置
  * 需要动态替换可执行代码（并且手动标注代码段可写属性）
  * 为了支持不同的tracer，需要有个框架

# 从编译开始

最根本的来说，ftrace的实现需要基于编译器的一个功能： fentry。也就是编译器会在每个函数执行开始前插入一个函数，通过这种方式来“捕捉”到每个函数的执行。

比如我们可以通过这个命令来生成一个反汇编代码，来观察打开ftrace选项时schedule_idle函数的汇编代码。

> make kernel/sched/core.s

此时我们观察反汇编中的schedule_idle函数可以看到：

```
schedule_idle:
1:      call    __fentry__
        .section __mcount_loc, "a",@progbits
```

也就是在调用schedule_idle函数时，真正执行函数本身代码前会调用__fentry__函数。而这个__fentry__函数的定义是

```
#ifdef CONFIG_DYNAMIC_FTRACE

SYM_FUNC_START(__fentry__)
	retq
SYM_FUNC_END(__fentry__)
EXPORT_SYMBOL(__fentry__)
```

注意这个是定义了动态ftrace的情况。

> **这就是ftrace最为本质的原理（之一）**

那为什么加了ftrace选项后就会多出这么一个函数呢？ 那是因为gcc支持一个特殊选项。我们可以通过下面的方式查看到内核编译是执行的命令。

```
cat kernel/sched/.core.o.cmd

gcc -pg -mrecord-mcount -mfentry ...
```

可以看到其中gcc命令行上添加了特殊的选项，以此来引入函数探针。

再进一步，内核为了能够处理这些探针，所有这些地址将会记录到__mcount_loc这个段。所以我们还要看一下链接描述文件：

```
/*
 * The ftrace call sites are logged to a section whose name depends on the
 * compiler option used. A given kernel image will only use one, AKA
 * FTRACE_CALLSITE_SECTION. We capture all of them here to avoid header
 * dependencies for FTRACE_CALLSITE_SECTION's definition.
 *
 * Need to also make ftrace_stub_graph point to ftrace_stub
 * so that the same stub location may have different protocols
 * and not mess up with C verifiers.
 */
#define MCOUNT_REC()	. = ALIGN(8);				\
			__start_mcount_loc = .;			\
			KEEP(*(__mcount_loc))			\
			KEEP(*(__patchable_function_entries))	\
			__stop_mcount_loc = .;			\
			ftrace_stub_graph = ftrace_stub;
```

这下我们放心了，散落在四海八荒的探针们，通过__start_mcount_loc就能找到了。

# 记录探针

就好像阎王爷的生死簿一样，散落在四海八荒的探针们也需要有人来统一管理。在ftrace中，使用了ftrace_page和dyn_ftrace来保存所有探针的位置。

这个过程发生在ftrace_process_locs()。

```
ftrace_init
    ftrace_process_locs(NULL, __start_mcount_loc, __stop_mcount_loc)
```

最后生成的样子如下面的示意图：

```
    ftrace_pages_start
      |
      v
    ftrace_page
    +-----------------------------+
    |index                        |
    |size                         |
    |    (int)                    |     array of dyn_ftrace
    |records                      |     +----------+----------+     +----------+----------+
    |    (struct dyn_ftrace*)     |---->|ip        |          | ... |          |          |
    |                             |     |flags     |          |     |          |          |
    |                             |     |arch      |          |     |          |          |
    |next                         |     +----------+----------+     +----------+----------+
    |    (struct ftrace_page*)    |
    +-----------------------------+
      |
      |
      v
    ftrace_page
    +-----------------------------+
    |index                        |
    |size                         |
    |    (int)                    |     array of dyn_ftrace
    |records                      |     +----------+----------+     +----------+----------+
    |    (struct dyn_ftrace*)     |---->|ip        |          | ... |          |          |
    |                             |     |flags     |          |     |          |          |
    |next                         |     |arch      |          |     |          |          |
    |    (struct ftrace_page*)    |     +----------+----------+     +----------+----------+
    +-----------------------------+
      |
      |
      v
    ftrace_page
    +-----------------------------+
    |index                        |
    |size                         |
    |    (int)                    |     array of dyn_ftrace
    |records                      |     +----------+----------+     +----------+----------+
    |    (struct dyn_ftrace*)     |---->|ip        |          | ... |          |          |
    |                             |     |flags     |          |     |          |          |
    |next                         |     |arch      |          |     |          |          |
    |    (struct ftrace_page*)    |     +----------+----------+     +----------+----------+
    +-----------------------------+
```

从此所有的探针就都在ftrace_pages_start为起始的一张表中。为了遍历这张特殊的表，访问到其中的每一个entry，就引入了这么一个宏定义：

```
/*
 * This is a double for. Do not use 'break' to break out of the loop,
 * you must use a goto.
 */
#define do_for_each_ftrace_rec(pg, rec)					\
	for (pg = ftrace_pages_start; pg; pg = pg->next) {		\
		int _____i;						\
		for (_____i = 0; _____i < pg->index; _____i++) {	\
			rec = &pg->records[_____i];
```

其中有两个for循环，一个是遍历ftrace_pages_start为首的链表，一个是编译每个ftrace_page的records。

# 替换成nop

在最开始我们看到了探针__fentry__的定义。虽然在动态ftrace的情况下，这个函数调用直接返回，但是毕竟每个函数都来这么以此跳转对内核的性能也是影响很大的。

为了尽量降低内核的开销，在动态ftrace的配置下，在启动时探针将会被替换为nop指令。这个过程就发生在记录完探针信息后。

```
ftrace_init
    ftrace_process_locs(NULL, __start_mcount_loc, __stop_mcount_loc)
        ftrace_update_code()
            ftrace_nop_initialize()
                ftrace_make_nop()
```

这层次真够深的，不过还没完，让我们再打开看看真正的细节。

```
ftrace_make_nop(mod, rec, MCOUNT_ADDR)
    old = ftrace_call_replace(ip, MCOUNT_ADDR)
        text_gen_insn(CALL_INSN_OPCODE, (void *)ip, (void *)MCOUNT_ADDR);
    new = ftrace_nop_replace
    ftrace_modify_code_direct(ip, old, new)
        ftrace_verify_code(ip, old)
        text_poke_early(ip, new, MCOUNT_INSN_SIZE)
```

简单来说就是找到旧的，拿到新的，替换。

这个text_gen_insn()很有意思，看了半天终于理解了。这是一个根据opcode来算出指令的函数，相当于手动执行了一次**编译的工作**。你仔细观察，这个函数的第三个参数其实就是__fentry__的地址，第二个参数就是探针插入的地址。而在计算指令的时候，第一个字节写入了opcode，而后面四个字节写入的是地址之间的差。而对于call指令来说，不就是要跳转到一个相对的地址么？所以说，text_gen_insn()函数手动计算出了探针插入某个函数后应该的模样。然后再把这个值传入到了ftrace_verify_code()，用来比较实际从内存中读出来的指令。如果一致，才能继续调用text_poke_early()来替换内核代码。

真的是煞费苦心啊。

# 动态替换 -- 替换谁

初始化完成之后，我们关心的是怎么样能够再次打开探针。首先我们得找到这种操作的入口，比如set_ftrace_filter文件。当然这又是一场艰苦卓绝的探索之旅。

首先我们找到这个文件的定义函数：

```
static const struct file_operations ftrace_filter_fops = {
	.open = ftrace_filter_open,
	.read = seq_read,
	.write = ftrace_filter_write,
	.llseek = tracing_lseek,
	.release = ftrace_regex_release,
};

trace_create_file("set_ftrace_filter", 0644, parent,
      ops, &ftrace_filter_fops);
```

当我们以写的模式打开这个文件时，会做一些准备工作，完成后大致形成这么一个结构：

```
      inode
      +--------------+
      |i_fop         |  = ftrace_filter_fops {.write = ftrace_filter_write}
      |i_private     |  = e.g. global_ops
      +--------------+


      struct file
      +--------------+    struct ftrace_iterator
      |.private_data |--->+------------+
      |              |    |.pg         |  = ftrace_pages_start
      +--------------+    |            |
                          |.ops        |  = global_ops
                          | ftrace_ops |
                          |            |
                          |            |  ftrace_func_entry
                          |.hash       |  (may copied from ftrace_ops->func_hash->[notrace_hash|filter_hash])
                          |            |  +----------+----------+----------+----------+
                          | ftrace_hash|  |          |          |          |          |
                          |            |  +----------+----------+----------+----------+
                          |            |
                          +------------+
```

其中出现了几个概念，ftrace_iterator, ftrace_ops和ftrace_hash。

别的我们先不管，先看这个ftrace_hash结构。其中保存的是本次我们想要改变的探针。

既然是写操作，那就要顺着ftrace_filter_write函数往下走，其中分支情况颇多。我们只看写入数字的情况。

```
match_records()
    add_rec_by_index(hash, )
        do_for_each_ftrace_rec(pg, rec) {
            if (pg->index <= index) {
                index -= pg->index;
                /* this is a double loop, break goes to the next page */
                break;
            }
            rec = &pg->records[index];
            enter_record(hash, rec, clear_filter);
            return 1;
        } while_for_each_ftrace_rec();
```

还记不记得这个do_for_each_ftrace_rec()?其实就是遍历了整个探针的记录，根据index找到究竟是要操作哪个探针。然后通过enter_record()这个这个探针添加到ftrace_hash这个哈希表中。

但是这样就结束了么？当然不是。到这里我们只是找到了一次修改时需要修改哪些探针地址，但是我们还没有真正的去修改代码呢。

那这部分隐藏在哪里？之前我们是从文件写操作ftrace_filter_write开始的，而这个秘密就隐藏在文件关闭的操作ftrace_regex_release()函数中。

# 动态替换 -- 改代码

在函数ftrace_regex_release()，对于写操作执行的重要任务通过ftrace_hash_move_and_update_ops()来完成。

```
ftrace_regex_release()
  ftrace_hash_move_and_update_ops()
    ftrace_hash_move()                (1)
    ftrace_ops_update_code()          (2)
```

也就是这个工作分成了两步

  * (1) 将ftrace_iterator->hash移动到ftrace_ops->func_hash上
  * (2) 真正的改动代码

第一步是将本地的record表同步到全局变量ftrace_ops中去，对这个数据类型我们将后续介绍。先来看真正执行代码改动的地方ftrace_ops_update_code()。

```
ftrace_ops_update_code
    ftrace_run_modify_code
        ftrace_run_update_code(FTRACE_UPDATE_CALLS)
            ftrace_arch_code_modify_prepare
                mutex_lock(&text_mutex);
                ftrace_poke_late = 1;
            arch_ftrace_update_code(FTRACE_UPDATE_CALLS)
                ftrace_replace_code()
            ftrace_arch_code_modify_post_process
                text_poke_finish();
                ftrace_poke_late = 0;
                mutex_unlock(&text_mutex);
```

这么看什么也没看出来，事实证明我们还要花一些耐心，需要继续探索ftrace_replace_code()。

```
ftrace_replace_code()
    ftrace_test_record(rec,), check dyn_ftrace.flags
        ftrace_check_record(rec, enable, false)
    old = ftrace_call_replace(rec->ip, ftrace_get_addr_curr(rec))
        ftrace_get_addr_curr(rec)
        text_gen_insn(CALL_INSN_OPCODE, rec->ip, ftrace_caller)
    ftrace_verify_code(rec->ip, old)
    new = ftrace_call_replace(rec->ip, ftrace_get_addr_new(rec))
        ftrace_get_addr_new(rec)
    text_poke_queue(rec->ip, new, MCOUNT_INSN_SIZE, NULL)
    ftrace_update_record(rec), update dyn_ftrace.flags
        ftrace_check_record(rec, enable, true)
    text_poke_finish()
        text_poke_bp_batch(tp_vec, tp_vec_nr), update instruction on live kernel
           text_poke(), the secret of modify kernel exec region
```

到这里，我们基本能看出一些端倪。

   * 找到旧的指令 -- old
   * 找到新的指令 -- new
   * 用text_poke()将old替换成new

而在作者slide[Ftrace Kernel Hooks: More than just tracing][1]中提到的使用断点来保证一致性的方法在text_poke_bp_batch()中，有兴趣的朋友可以进去看，注释写得很清楚了。

最后的text_poke()也是一个非常有意思的东西。大家都知道内核代码是只读的，为了能够对代码有写的能力，另起炉灶的增加了一个mm_struct结构体，将需要修改的代码mapping到页表中。然后切换到这个临时的页表。这个还是很神奇的。

到这里，我们终于弄懂了ftrace最基本的原理：

   * 插入探针函数
   * 动态修改探针

当然这一切仅仅是开始。

[1]: https://blog.linuxplumbersconf.org/2014/ocw/system/presentations/1773/original/ftrace-kernel-hooks-2014.pdf
