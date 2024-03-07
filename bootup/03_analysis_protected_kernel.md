进入保护模式内核后，还有很多和体系结构相关的工作需要准备。
下面我们就利用bochs的调试功能，对这部分代码作一些分析。

具体如何进入到这部分调试界面，可参考[start_kernel之前][1]

# 计算当前内核被加载的地址

进入保护模式内核后，有这么一段代码是用来计算保护模式内核被加载到哪里，并把这个地址保存到epb中。
arch/x86/boot/compressed/head_64.S

```
	leal	(BP_scratch+4)(%esi), %esp
	call	1f
1:	popl	%ebp
	subl	$ rva(1b), %ebp
```

原理是call的短调用，会将下一条指令的地址压栈。也就是popl %ebp的地址会被压栈。而rva(1b)是这条指令相对于startup_32的偏移，所以两者相减就得到了本次运行个过程中，实际被加载到的地址。

现在我们就用bochs来验证一下，看看保护模式的内核是不是加载到了0x100000。

```
(0) Breakpoint 2, 0x0000000000100000 in ?? ()
Next at t=90656425
(0) [0x0000000000100000] 0010:0000000000100000 (unk. ctxt): cld                       ; fc
<bochs:7> u /10
00100000: (                    ): cld                       ; fc
00100001: (                    ): cli                       ; fa
00100002: (                    ): lea esp, dword ptr ds:[esi+488] ; 8da6e8010000
00100008: (                    ): call .+0                  ; e800000000
0010000d: (                    ): pop ebp                   ; 5d
0010000e: (                    ): sub ebp, 0x0000000d       ; 83ed0d
```

在0x100000处断点停止后，先看一下反汇编。popl %ebp指令的地址是0x10000d。而这条指令的相对地址是0x0d。接下来我们来确认压栈的地址，和最后计算的结果。

```
<bochs:12> 
Next at t=90656430
(0) [0x000000000010000e] 0010:000000000010000e (unk. ctxt): sub ebp, 0x0000000d       ; 83ed0d
<bochs:13> r
rax: 0x00000000_00100000 rcx: 0x00000000_00000000
rdx: 0x00000000_00000000 rbx: 0x00000000_00000000
rsp: 0x00000000_00014338 rbp: 0x00000000_0010000d
...
```

在执行减法前，查看一下寄存器。发现ebp的值已经是0x10000d了。

```
<bochs:14> n
Next at t=90656431
(0) [0x0000000000100011] 0010:0000000000100011 (unk. ctxt): lea eax, dword ptr ss:[ebp+10657808] ; 8d8510a0a200
<bochs:15> r
rax: 0x00000000_00100000 rcx: 0x00000000_00000000
rdx: 0x00000000_00000000 rbx: 0x00000000_00000000
rsp: 0x00000000_00014338 rbp: 0x00000000_00100000
```

接下去再执行一条，也就是减去0x0d，ebp的值就是0x100000了。这就是我们这次保护模式内核被加载的地址，也是我们想要的。

我后来想到，这么计算还需要依赖一个条件，就是代码段的基址需要是0。否则计算出的值需要加上基址才是。那就来看看当前的代码段设置。

```
<bochs:16> info gdt
Global Descriptor Table (base=0x0000000000013c50, limit=39):
GDT[0x00]=??? descriptor hi=0x00000000, lo=0x00000000
GDT[0x01]=??? descriptor hi=0x00000000, lo=0x00000000
GDT[0x02]=Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, Accessed, 32-bit
GDT[0x03]=Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
GDT[0x04]=32-Bit TSS (Busy) at 0x00001000, length 0x00067
You can list individual entries with 'info gdt [NUM]' or groups with 'info gdt [NUM] [NUM]'
```

看来确实是0。

# 切换GDT

紧接着内核切换了gdt.

```
	/* Load new GDT with the 64bit segments using 32bit descriptor */
	leal	rva(gdt)(%ebp), %eax
	movl	%eax, 2(%eax)
	lgdt	(%eax)
```

而这里的gdt定义是

```
SYM_DATA_START_LOCAL(gdt)
	.word	gdt_end - gdt - 1
	.long	0
	.word	0
	.quad	0x00cf9a000000ffff	/* __KERNEL32_CS */
	.quad	0x00af9a000000ffff	/* __KERNEL_CS */
	.quad	0x00cf92000000ffff	/* __KERNEL_DS */
	.quad	0x0080890000000000	/* TS descriptor */
	.quad   0x0000000000000000	/* TS continued */
SYM_DATA_END_LABEL(gdt, SYM_L_LOCAL, gdt_end)
```

所以，gdt这块区域的开头几个字节是用作gdtr的,而且16位的界限已经定义好了。但是基址没有定义，因为运行时的地址我们事先不知道。但是还记得我们刚才分析的代码么？对了，就是后把当前加载的地址保存到了ebp。所以这第一句 leal rva(gdt)(%ebp), %eax就是计算了gdt的基址。

分析完了，那就用bochs调试一下看看。我们在leal这句执行完后，查看一下。

```
<bochs:24> r
rax: 0x00000000_00b2a010 rcx: 0x00000000_00000000
rdx: 0x00000000_00000000 rbx: 0x00000000_00000000
...
```

说明gdt的地址在0xb2a010，我们用内存查看工具查看一下。

```
<bochs:26> xp /4xw 0x00b2a010
[bochs]:
0x0000000000b2a010 <bogus+       0>:	0x0000002f	0x00000000	0x0000ffff	0x00cf9a00
```

其中0x2f是界限，意思是gdt的大小是0x2f + 0x01 = 0x30 也就是有48字节。而我们gdt中，正好占了48字节，符合！
再来看dump出来的第二块8字节，因为x86是little endian的，所以实际值就是0x00cf9a000000ffff。这个不正好和__KERNEL32_CS所应以的值一样吗？完美！

执行完movl	%eax, 2(%eax)后，我们再查看一下内存。因为gdt的最开始两个字节是界限，我们跳过这部分，直接dump后面的四个字节。

```
<bochs:33> xp /xw 0x00b2a012
[bochs]:
0x0000000000b2a012 <bogus+       0>:	0x00b2a010
```

显示此时这部分内存的内容已经是我们刚才所得到的gdt基址0x00b2a010了。

切换gdt的前后，我们都查看一下gdt。
```
<bochs:35> info gdt
Global Descriptor Table (base=0x0000000000013c50, limit=39):
GDT[0x00]=??? descriptor hi=0x00000000, lo=0x00000000
GDT[0x01]=??? descriptor hi=0x00000000, lo=0x00000000
GDT[0x02]=Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, Accessed, 32-bit
GDT[0x03]=Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
GDT[0x04]=32-Bit TSS (Busy) at 0x00001000, length 0x00067
<bochs:36> n
Next at t=109056323
(0) [0x000000000010001d] 0010:000000000010001d (unk. ctxt): mov eax, 0x00000018       ; b818000000
<bochs:37> info gdt
Global Descriptor Table (base=0x0000000000b2a010, limit=47):
GDT[0x00]=??? descriptor hi=0x000000b2, lo=0xa010002f
GDT[0x01]=Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, 32-bit
GDT[0x02]=Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, 64-bit
GDT[0x03]=Data segment, base=0x00000000, limit=0xffffffff, Read/Write
GDT[0x04]=32-Bit TSS (Available) at 0x00000000, length 0x00000
GDT[0x05]=??? descriptor hi=0x00000000, lo=0x00000000
```

这里可以看到，gdt确实发生了变化。大功告成！

[1]: /bootup/02_before_start_kernel.md