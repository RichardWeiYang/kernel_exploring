在x86平台上，专门有一张表来保存中断和异常处理对应的函数。这张表就叫做IDT(Interrupt Descriptor Table)。

# IDT的长相

先来看手册上的一张图，这张图是在SDM Volume 3中的Figure 2-1. IA-32 System-Level Registers and Data Structures。

![IA-32 System-Level Registers and Data Structures](/interrupt_exception/system_level_registers.png)

在这张图中， IDT位于最左边靠近下方的部分。和GDT/LDT类似，IDT的位置由IDTR寄存器保存。所以在代码中加载IDT的代码如下：

```
static inline void native_load_idt(const struct desc_ptr *dtr)
{
	asm volatile("lidt %0"::"m" (*dtr));
}
```

而其中struct desc_ptr的一个实现则长成这个样子：

```
/* Must be page-aligned because the real IDT is used in a fixmap. */
gate_desc idt_table[IDT_ENTRIES] __page_aligned_bss;

struct desc_ptr idt_descr __ro_after_init = {
	.size		= (IDT_ENTRIES * 2 * sizeof(unsigned long)) - 1,
	.address	= (unsigned long) idt_table,
};
```

嗯，这下算是把代码和手册在这个点上结合起来了。

# IDT的基本工作过程

具体的过程是有点复杂的，那在这里对我而言只要知道一下这点暂时就足够用了。

> CPU在执行下一条指令之前，将会检查此时是否有中断或者异常发生。如果有，则会判断中断或异常的vector number。在做完一系列检查、保护工作之后，会执行**IDT中对应vector number**的处理函数。

具体这个vector number是如何得到的，又做了哪些检查和保护，不在本次学习的重点中。关键是我们知道了硬件的IDT和软件代码之间的关系。

# 内核中IDT的布局

说的这么眼花缭乱，那我们来看看内核中IDT是个什么样子吧。这个呢就在内核代码arch/x86/include/asm/irq_vectors.h中了。

```
/*
 * Linux IRQ vector layout.
 *
 * There are 256 IDT entries (per CPU - each entry is 8 bytes) which can
 * be defined by Linux. They are used as a jump table by the CPU when a
 * given vector is triggered - by a CPU-external, CPU-internal or
 * software-triggered event.
 *
 * Linux sets the kernel code address each entry jumps to early during
 * bootup, and never changes them. This is the general layout of the
 * IDT entries:
 *
 *  Vectors   0 ...  31 : system traps and exceptions - hardcoded events
 *  Vectors  32 ... 127 : device interrupts
 *  Vector  128         : legacy int80 syscall interface
 *  Vectors 129 ... INVALIDATE_TLB_VECTOR_START-1 except 204 : device interrupts
 *  Vectors INVALIDATE_TLB_VECTOR_START ... 255 : special interrupts
 *
 * 64-bit x86 has per CPU IDT tables, 32-bit has one shared IDT table.
 *
 * This file enumerates the exact layout of them:
 */
```

# 心中的疑惑

到这里，虽然已经对IDT有了大致的了解，但是我想你一定也和我一样还有很多疑惑。

* 中断和异常有区别么？
* 中断和异常里究竟有什么？

不着急，让我们一点点揭开这神秘的面纱。
