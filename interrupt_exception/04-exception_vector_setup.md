知道了IDT中前32个中断向量用来处理异常后，我就很想知道这些异常向量对应的IDT项是如何初始化的，如何对应到异常处理函数的。

这一小节我们就来解开这部分的谜团。

# 全局观

先来看看内核启动时，是在哪里初始化的异常向量表。

```
  start_kernel()
    trap_init()
      idt_setup_traps()
        idt_setup_from_table(idt_table, def_idts)
```

上面的流程中，基本看出了异常向量表初始化的位置。进一步从代码中可以看出，实际的工作就是把def_idts中的内容写到idt_table对应的异常向量中。

# 从def_idts开始

既然是将def_idts写到idt_table，那就来看看这个表的内容。

```
/* Interrupt gate */
#define INTG(_vector, _addr)				\
	G(_vector, _addr, DEFAULT_STACK, GATE_INTERRUPT, DPL0, __KERNEL_CS)

static const __initconst struct idt_data def_idts[] = {
	INTG(X86_TRAP_DE,		divide_error),
  ...
};
```

可以看到，这张表中的一项对应了一个异常处理。其中_addr就是异常处理函数了。

# idtentry和异常处理函数

接着我们就要找到这个divide_error异常处理函数的定义了。开始我怎么也找不到，后来才发现这个divide_error的异常处理函数是在汇编代码中用idtentry来实现的。

```
idtentry divide_error			do_divide_error			has_error_code=0

.macro idtentry sym do_sym has_error_code:req paranoid=0 shift_ist=-1
ENTRY(\sym)
	UNWIND_HINT_IRET_REGS offset=\has_error_code*8

  ...

	call	\do_sym

  ...

END(\sym)
.endm
```

省略众多细节，突出大致结构。idtentry为每一个异常处理做了基本统一的处理，然后对应不同的异常再调用do_sym函数处理。对应divide_error，这个函数就是do_divide_error。

# DO_ERROR和信号

内核为了代码简洁和统一，也用了一个宏DO_ERROR来定义统一的异常处理方式。

```
#define DO_ERROR(trapnr, signr, sicode, addr, str, name)		   \
dotraplinkage void do_##name(struct pt_regs *regs, long error_code)	   \
{									   \
	do_error_trap(regs, error_code, str, trapnr, signr, sicode, addr); \
}

DO_ERROR(X86_TRAP_DE,     SIGFPE,  FPE_INTDIV,   IP, "divide error",        divide_error)
```

从上面的代码片段可以看出，大家殊途同归，异常处理最后都走到了do_error_trap()函数，而这个函数最后又调用了do_trap()。

```
static void
do_trap(int trapnr, int signr, char *str, struct pt_regs *regs,
	long error_code, int sicode, void __user *addr)
{
	struct task_struct *tsk = current;


	if (!do_trap_no_signal(tsk, trapnr, str, regs, error_code))
		return;

	show_signal(tsk, signr, "trap ", str, regs, error_code);

	if (!sicode)
		force_sig(signr, tsk);
	else
		force_sig_fault(signr, sicode, addr, tsk);
}
```

> **所以异常处理的最后就是通过内核向该进程发送一个信号，由进程捕获该信号来处理。**

从代码可以看到这个信号的值是signr，一路追踪divide_error对应的信号在DO_ERROR宏中定义为SIGFPE。到这里我们基本理清了异常向量初始化的内容，以及异常处理函数采用信号和进程通信。至于内核如何产生和发送信号，进程如何处理信号，那又是一个值得探索的话题了。
