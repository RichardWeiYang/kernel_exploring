进入保护模式内核后，还有很多和体系结构相关的工作需要准备。
下面我们就利用bochs的调试功能，对这部分代码作一些分析。

具体如何进入到这部分调试界面，可参考[start_kernel之前][1]

# 计算当前内核被加载的地址

进入保护模式内核后，有这么一段代码是用来计算保护模式内核被加载到哪里，并把这个地址保存到epb中。

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

[1]: /bootup/02_before_start_kernel.md