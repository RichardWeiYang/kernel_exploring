系统调用是用户程序和内核之间沟通的桥梁，没有他内核跑起来也根本用不上。不过我们好像从来就没有正眼看过他。今天我们就来瞧瞧这位一直默默付出的幕后英雄。

# 从调用方式开始

以前在课本上我们学到系统调用是通过int 0x80来实现的。也就是用户程序通过IDT 0x80的这个中断向量和内核发生联系。只不过那已经是很久以前的事情了。温故而知新，那就从新旧两种调用方式入手吧。

## 从前的调用方式

```
.data                   # section declaration

msg:
	.string "Hello, world!\n"
	len = . - msg   # length of our dear string

.text                   # section declaration

                        # we must export the entry point to the ELF linker or
  .global main          # loader. They conventionally recognize _start as their
                        # entry point. Use ld -e foo to override the default.

main:

# write our string to stdout

	movl    $len,%edx   # third argument: message length
	movl    $msg,%ecx   # second argument: pointer to message to write
	movl    $1,%ebx     # first argument: file handle (stdout)
	movl    $4,%eax     # system call number (sys_write)
	int     $0x80       # call kernel

# and exit

	movl    $0,%ebx     # first argument: exit code
	movl    $1,%eax     # system call number (sys_exit)
	int     $0x80       # call kernel
```

可以看到，这种方式就是通过int 0x80来实现的。这段代码在x86平台上还能运行，用如下方式进行编译：

```
# gcc -c hello-ia32.s
# ld -e main -o hell hello-ia32.o
```

## 现在的调用方式

接下来看看新的方式：

```
.data

msg:
	.ascii "Hello World!\n"
	len = . - msg

.text
	.global _start

_start:
	movq $1, %rax
	movq $1, %rdi
	movq $msg, %rsi
	movq $len, %rdx
	syscall

	movq $60, %rax
	xorq %rdi, %rdi
	syscall
```

怎么样，看着是不是没啥大区别？关键是这里使用的是syscall这个指令来完成系统调用。既然如此，那就从这个指令开始吧。

# 从syscall到sys_call_table

了解了当前系统调用的实现方式，接着我们就想要了解它是如何同内核中的系统调用函数结合的，如何一步步调用到那些熟悉的系统调用函数的。

想要了解syscall这个指令这一切还要从手册开始。在SDM Volume 2中就有这个指令的详尽解释，这里摘抄一段。

> SYSCALL invokes an OS system-call handler at privilege level 0. It does so by loading RIP from the IA32_LSTAR MSR (after saving the address of the instruction following SYSCALL into RCX).

原来这次不是通过IDT找到跳转地址，而是通过了一个msr来保存。而这个msr在内核中定义为 MSR_LSTAR。由此我们找到了内核中这段代码：

```
  wrmsrl(MSR_LSTAR, (unsigned long)entry_SYSCALL_64);
```

在entry_SYSCALL_64这段汇编中，又看到了这么一行醒目的代码：

```
  call	do_syscall_64		/* returns with IRQs disabled */
```

当我们进一步打开这个函数时，一切或许就有些明朗了：

```
__visible void do_syscall_64(unsigned long nr, struct pt_regs *regs)
{
	struct thread_info *ti;

	enter_from_user_mode();
	local_irq_enable();
	ti = current_thread_info();
	if (READ_ONCE(ti->flags) & _TIF_WORK_SYSCALL_ENTRY)
		nr = syscall_trace_enter(regs);

	/*
	 * NB: Native and x32 syscalls are dispatched from the same
	 * table.  The only functional difference is the x32 bit in
	 * regs->orig_ax, which changes the behavior of some syscalls.
	 */
	nr &= __SYSCALL_MASK;
	if (likely(nr < NR_syscalls)) {
		nr = array_index_nospec(nr, NR_syscalls);
		regs->ax = sys_call_table[nr](regs);
	}

	syscall_return_slowpath(regs);
}
```

在函数中最后一段的if语句中，可以看到我们通过nr索引了sys_call_table的相关项并调用。

这就是我们要找的**系统调用函数表**了。

由此我们一路走来，终于找到了目标。在接着往下探索之前，先来回顾一下我们是怎么走到这里的。

> syscall -> entry_SYSCALL_64 -> do_syscall_64 -> sys_call_table

# sys_call_table的构造

走到了这，基本上万里长征完成了大半。接下来就是查看这张表是如何构成的。

这个就是sys_call_table的定义了。

```
asmlinkage const sys_call_ptr_t sys_call_table[__NR_syscall_max+1] = {
	/*
	 * Smells like a compiler bug -- it doesn't work
	 * when the & below is removed.
	 */
	[0 ... __NR_syscall_max] = &sys_ni_syscall,
#include <asm/syscalls_64.h>
};
```

这个定义中的关键就在最后那个包含的头文件中，而这个文件是编译时生成的

> ./arch/x86/include/generated/asm/syscalls_64.h

生成的规则是

```
syscall64 := $(srctree)/$(src)/syscall_64.tbl
systbl := $(srctree)/$(src)/syscalltbl.sh

quiet_cmd_systbl = SYSTBL  $@
      cmd_systbl = $(CONFIG_SHELL) '$(systbl)' $< $@

$(out)/syscalls_64.h: $(syscall64) $(systbl)
	$(call if_changed,systbl)
```

如果可以用一个为代码表示这个过程的话，那就是

> syscalls_64.h = syscalltbl.sh (syscall_64.tbl)

所以这个头文件syscalls_64.h是脚本syscalltbl.sh处理文件syscall_64.tbl的结果。

当你打开syscall_64.tbl的时候，会有种云开雾散的感觉。

```
#
# 64-bit system call numbers and entry vectors
#
# The format is:
# <number> <abi> <name> <entry point>
#
# The __x64_sys_*() stubs are created on-the-fly for sys_*() system calls
#
# The abi is "common", "64" or "x32" for this file.
#
0	common	read			__x64_sys_read
1	common	write			__x64_sys_write
2	common	open			__x64_sys_open
3	common	close			__x64_sys_close
```

这里就保存这整个系统调用的对应关系。

看到了生成头文件的原料，那来看看生成的头文件是什么样子。打开生成的syscalls_64.h，看到syscall_64.tbl中的每一行都展开成如下的形式：

```
#ifdef CONFIG_X86
__SYSCALL_64(0, __x64_sys_read, )
#else /* CONFIG_UML */
__SYSCALL_64(0, sys_read, )
#endif
```

而这个__SYSCALL_64的宏定义如下：

```
#define __SYSCALL_64(nr, sym, qual) [nr] = sym,
```

这样可能还不是很清楚，我们再把这个代入到sys_call_table的定义中再看一眼。

```
asmlinkage const sys_call_ptr_t sys_call_table[__NR_syscall_max+1] = {
    [0 ... __NR_syscall_max] = &sys_ni_syscall,
    [0] = __x64_sys_read,
    [1] = __x64_sys_write,
    [2] = __x64_sys_open,
    ...
    ...
    ...
};
```

# 最后一环

表也有了，现在就差最后一环了，函数__x64_sys_read()在哪里？这事我们还得倒过来看。

正着找找不到，那我们就反过来找。从read系统调用的定义入手。

```
SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count)
{
	return ksys_read(fd, buf, count);
}
```

这其中有个宏SYSCALL_DEFINE3。

```
#define SYSCALL_DEFINE3(name, ...) SYSCALL_DEFINEx(3, _##name, __VA_ARGS__)

#define SYSCALL_DEFINEx(x, sname, ...)				\
	SYSCALL_METADATA(sname, x, __VA_ARGS__)			\
	__SYSCALL_DEFINEx(x, sname, __VA_ARGS__)
```

最后的最后，还有一个__SYSCALL_DEFINEx。

```
#define __SYSCALL_DEFINEx(x, name, ...)					\
	asmlinkage long __x64_sys##name(const struct pt_regs *regs);	\
	ALLOW_ERROR_INJECTION(__x64_sys##name, ERRNO);			\
	static long __se_sys##name(__MAP(x,__SC_LONG,__VA_ARGS__));	\
	static inline long __do_sys##name(__MAP(x,__SC_DECL,__VA_ARGS__));\
	asmlinkage long __x64_sys##name(const struct pt_regs *regs)	\
	{								\
		return __se_sys##name(SC_X86_64_REGS_TO_ARGS(x,__VA_ARGS__));\
	}								\
	__IA32_SYS_STUBx(x, name, __VA_ARGS__)				\
	static long __se_sys##name(__MAP(x,__SC_LONG,__VA_ARGS__))	\
	{								\
		long ret = __do_sys##name(__MAP(x,__SC_CAST,__VA_ARGS__));\
		__MAP(x,__SC_TEST,__VA_ARGS__);				\
		__PROTECT(x, ret,__MAP(x,__SC_ARGS,__VA_ARGS__));	\
		return ret;						\
	}								\
	static inline long __do_sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))
```

这玩意确实有点长，不过你是不是看到了__x64_sys##name的字样呢？

好了，我相信你已经懂了。在sys_call_table表中填入的函数入口，就是由__SYSCALL_DEFINEx定义出来的。

经过了一番探索，我们终于理清了系统调用在linux中是如何定义和关联的。
