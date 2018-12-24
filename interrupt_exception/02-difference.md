在学习的初期，我一直有这么个疑惑：

> 不都是在IDT中么？不都是通过vector number索引么？中断和异常到底有啥区别呢？

这次我们就来尝试揭开这个谜团。

# 雷锋和雷峰塔

> 在想究竟用哪个标题来描述这两者之间的区别，突然想到之前看到过这个就用上了。

当然，实际上这两者的差别并没有这么大。我能想到的两点区别是：

* 来源不同
* 处理不同
* 位置不同

来源不同是指中断是外界引入的。比如键盘，硬盘，网络的数据相应是中断，而异常则是软件执行时自己触发的，比如除0，溢出。
在处理上，异常通常是发送一个信号给到执行的进程，由进程自行决定该如何处理。而中断的处理则由中断函数在处理。
位置不同写的有点模糊，意思是在IDT中，异常和nmi中断的中断向量是固定的，而可屏蔽中断的中断向量是写死的。

来源的不同就不用我来讲了，这是个事实。我就简单说一下后两者。

# 处理不同

总的来说，中断和异常的处理在内核中是分成两套机制完成的。

* 异常的处理最后落到给进程发送信号上
* 中断的处理最后落到do_IRQ函数上

具体的细节将在后续的小节中展开。

# 对应的中断向量

除此之外，中断和异常在IDT中的位置也是有讲究的。

* 异常和nmi的中断向量是固定的
* 可屏蔽中断的中断向量则是可配的

在上一节中，我们看到了中断向量的整个布局。现在我们进一步仔细看看其中哪些是中断哪些是异常。
下面的描述在文件arch/x86/include/asm/irq_vectors.h中。

```
*  Vectors   0 ...  31 : system traps and exceptions - hardcoded events
*  Vectors  32 ... 127 : device interrupts
*  Vector  128         : legacy int80 syscall interface
*  Vectors 129 ... INVALIDATE_TLB_VECTOR_START-1 except 204 : device interrupts
*  Vectors INVALIDATE_TLB_VECTOR_START ... 255 : special interrupts
```

可以看到其中大部分是中断，而在0-31中保存的是异常的中断向量。那进一步再来看异常向量的布局。

```
/* Interrupts/Exceptions */
enum {
	X86_TRAP_DE = 0,	/*  0, Divide-by-zero */
	X86_TRAP_DB,		/*  1, Debug */
	X86_TRAP_NMI,		/*  2, Non-maskable Interrupt */
	X86_TRAP_BP,		/*  3, Breakpoint */
	X86_TRAP_OF,		/*  4, Overflow */
	X86_TRAP_BR,		/*  5, Bound Range Exceeded */
	X86_TRAP_UD,		/*  6, Invalid Opcode */
	X86_TRAP_NM,		/*  7, Device Not Available */
	X86_TRAP_DF,		/*  8, Double Fault */
	X86_TRAP_OLD_MF,	/*  9, Coprocessor Segment Overrun */
	X86_TRAP_TS,		/* 10, Invalid TSS */
	X86_TRAP_NP,		/* 11, Segment Not Present */
	X86_TRAP_SS,		/* 12, Stack Segment Fault */
	X86_TRAP_GP,		/* 13, General Protection Fault */
	X86_TRAP_PF,		/* 14, Page Fault */
	X86_TRAP_SPURIOUS,	/* 15, Spurious Interrupt */
	X86_TRAP_MF,		/* 16, x87 Floating-Point Exception */
	X86_TRAP_AC,		/* 17, Alignment Check */
	X86_TRAP_MC,		/* 18, Machine Check */
	X86_TRAP_XF,		/* 19, SIMD Floating-Point Exception */
	X86_TRAP_IRET = 32,	/* 32, IRET Exception */
};
```

这个表大家也可以对应SDM Volume 3中Table 6-1 Protected-Mode Exceptions and Interrupts一起看。这样也找到了手册和代码之间的一个对应关系。

好了，了解了这两者的区别后，我们就可以来看看内核中是如何处理他们的了。
