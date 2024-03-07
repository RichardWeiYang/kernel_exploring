在进入start_kernel之前，那真的是一片黑暗的路程。因为好多都是用汇编写的，对我来说简直就是抓瞎。

在[bzImage的全貌][1]中我们看过了make install时安装的bzImage的组成部分。而start_kernel在这几个组成部分的最后一点。


```
                                     *
                                     | <-- vmlinux.lds.S
                                     |
                                   vmlinux
                                     |
                                     |  arch/x86/boot/compressed/*
       arch/x86/boot/*                \  /
              |                        \/
              | <-- setup.ld            | <-- vmlinux.lds
              |                         |
              |                         v
              |              arch/x86/boot/compressed/vmlinux 
              |                         |
              |                         | <-- objcopy
              |                         |
              v                         v
    arch/x86/boot/setup.bin  arch/x86/boot/vmlinux.bin  
                   \         /
                    \       /
               arch/x86/boot/bzImage
```

这次我们就来看看被安装的内核是通过哪些步骤走到start_kernel的。

从代码上看，内核加载后到start_kernel前经历了下面的步骤。

**setup.bin**:
```
_start -> main -> go_to_protected_mode -> protected_mode_jump(boot_params.hdr.code32_start, )
```

**vmlinux.bin**:
```
startup_32 -> startup_64 -> extract_kernel ->
```

**vmlinux**:
```
startup_64 -> initial_code -> x86_64_start_kernel -> x86_64_start_reservations
```

但是真的是这样吗？如何可以确认呢？好了，那就请出bochs模拟器。来确认一下整个流程吧。

# 用bochs探索start_kernel之前的黑暗世界

bochs是一个x86的模拟器，据说还能运行win98。而且调试友好，在《自己动手写操作系统》一书中就是用bochs来运行手写的操作系统的。这里我们就要再次请出它来帮助我们了解start_kernel之前的黑暗世界。

## 安装bochs

```
sudo apt install bochs
sudo apt install bochs-x
```

在ubuntu上运行这两个命令就能安装bochs了。记得安装bochs-x，否则会报错。

## 准备启动镜像

其实x86内核编译里有制作启动盘的目标，包括了软盘、光盘、硬盘。这里我们只用光盘。

```
make isoimage
```

另外记得安装个依赖

```
sudo apt install syslinux_utils
sudo apt install isolinux
```

执行这条命令就可以生成arch/x86/boot/image.iso启动光盘，其中包含了当前目录编译出的最新kernel。
但是这个命令有个问题，不确定是不是内核开发遗漏了，一定要加上这个改动才能制作成功。

```
diff --git a/arch/x86/boot/Makefile b/arch/x86/boot/Makefile
index 3cece19b7473..8b178eded5bc 100644
--- a/arch/x86/boot/Makefile
+++ b/arch/x86/boot/Makefile
@@ -117,7 +117,7 @@ $(obj)/compressed/vmlinux: FORCE
 # bzdisk/fdimage/hdimage/isoimage kernel
 FDARGS =
 # Set this if you want one or more initrds included in the image
-FDINITRD =
+FDINITRD = /boot/initrd.img-6.0.0-rc4yw+
 
 imgdeps = $(obj)/bzImage $(obj)/mtools.conf $(src)/genimage.sh
```

就是一定要指定根文件才能制作。等有空了我问问内核社区这是几个意思。

## bochs配置文件

有了镜像，bochs也装好了，接下来我们就可以启动了。

可以参考下面的配置，

```
###############################################################
# Configuration file for Bochs
###############################################################

# how much memory the emulated machine will have
megs: 128

# filename of ROM images
romimage: file=/usr/share/bochs/BIOS-bochs-latest
vgaromimage: file=/usr/share/vgabios/vgabios.bin

# what disk images will be used
ata0-slave:  type=cdrom, path="image.iso", status=inserted

# choose the boot disk.
boot: cdrom

# where do we send log messages?
# log: bochsout.txt

# disable the mouse
mouse: enabled=0

# enable key mapping, using US layout as default.
#keyboard_mapping: enabled=1, map=/usr/share/bochs/keymaps/x11-pc-us.map
```

我是把image.iso放在配置文件同一个目录的，大家可以根据自己习惯调整。

然后，启动，运行！

```
bochs -f bochsrc
```

## 内核加载到了哪里？

一切的都很顺利？但是说好的调试呢？断点在哪里设置？什么时候进入的保护模式？

好像我们什么都不知道。我们想要的是**内核究竟被加载到哪里了**。这样我们才能设置断点，然后调试查看。

这时候我突然想到了内核文档，说不定文档里会有写呢？别说，我还真找到一个文档[boot.rst][3]。人是这么说的：

```
For a modern bzImage kernel with boot protocol version >= 2.02, a
memory layout like the following is suggested::

		~                        ~
		|  Protected-mode kernel |
	100000  +------------------------+
		|  I/O memory hole	 |
	0A0000	+------------------------+
		|  Reserved for BIOS	 |	Leave as much as possible unused
		~                        ~
		|  Command line		 |	(Can also be below the X+10000 mark)
	X+10000	+------------------------+
		|  Stack/heap		 |	For use by the kernel real-mode code.
	X+08000	+------------------------+
		|  Kernel setup		 |	The kernel real-mode code.
		|  Kernel boot sector	 |	The kernel legacy boot sector.
	X       +------------------------+
		|  Boot loader		 |	<- Boot sector entry point 0000:7C00
	001000	+------------------------+
		|  Reserved for MBR/BIOS |
	000800	+------------------------+
		|  Typically used by MBR |
	000600	+------------------------+
		|  BIOS use only	 |
	000000	+------------------------+
```

实模式的内核加载地址是个X，这有点头大。那究竟是哪里呢？

回忆一下《自己动手写操作系统》，在开机上电到内核运行经历了这么几个步骤：

* 系统先运行BIOS
* 由BIOS找到boot sector
* boot sector加载loader
* loader加载内核，并跳转

所以在内核运行前，还有几个步骤要执行。在我们制作出的启动镜像里，这个loader是syslinux完成的。还记得我们制作镜像时安装的依赖么？具体可以看arch/x86/boot/genimage.sh中geniso函数。想进一步了解syslinux的，可以参考[Syslinux Tutorial][2]。

所以决定内核加载到哪里的，是syslinux决定的。既然都是开源代码，那就。。。看代码吧。

在此跳过细节，直接给出结果。在我编译的内核情况下，实模式内核加载到了**0x10000**，保护模式内核加载到了**0x100000**。想要看看syslinux的，可以在[syslinux][4]上找到我定位内核加载地址的代码。

## 确认内核加载地址

知道了内核加载到哪里，我们就可以在对应的地址设置断点，来确认这个发现是不是真的。

```
pb 0x10000
pb 0x100000
```

但是一直看不到停在实模式内核代码上，这是为什么呢？看了代码想起来了，原来实模式内核的第一个扇区是一个引导盘。实际有功效的代码是在512字节后。所以syslinux是直接条到这里开始的么？

那我们就把断点调整以下，看看效果。

```
pb 0x10200
pb 0x100000
```

怎么样，当你看到在断点停下来的时候，是不是很激动人心！（断点有两次会停在bootloader里，所以前两次的忽略。）

下面上调试的实际结果，来感受以下。

### 反汇编实模式内核代码

```
(0) Breakpoint 1, 0x0000000000010200 in ?? ()
Next at t=135209726
```

实模式内核加载地址+偏移512的断点触发了。确认当前地址是0x10200。

```
<bochs:7> creg
CR0=0x60000010: pg CD NW ac wp ne ET ts em mp pe
CR2=page fault laddr=0x0000000000000000
CR3=0x0000000000000000
    PCD=page-level cache disable=0
    PWT=page-level write-through=0
CR4=0x00000000: smep osxsave pcid fsgsbase smx vmx osxmmexcpt osfxsr pce pge mce pae pse de tsd pvi vme
CR8: 0x0
EFER=0x00000000: ffxsr nxe lma lme sce
```

查看当前寄存器，确认目前在实模式，CR0的pe是小写。页表也没有打开pg是小写。

```
<bochs:9> u /5
00010200: (                    ): jmp .+106                 ; eb6a
00010202: (                    ): dec ax                    ; 48
00010203: (                    ): jb .+83                   ; 647253
00010206: (                    ): lar ax, word ptr ds:[bx+si] ; 0f0200
00010209: (                    ): add byte ptr ds:[bx+si], al ; 0000
<bochs:10> n
Next at t=135209727
(0) [0x000000000001026c] 1020:006c (unk. ctxt): mov ax, ds                ; 8cd8
<bochs:11> u /20
0001026c: (                    ): mov ax, ds                ; 8cd8
0001026e: (                    ): mov es, ax                ; 8ec0
00010270: (                    ): cld                       ; fc
00010271: (                    ): mov dx, ss                ; 8cd2
00010273: (                    ): cmp dx, ax                ; 39c2
00010275: (                    ): mov dx, sp                ; 89e2
00010277: (                    ): jz .+22                   ; 7416
00010279: (                    ): mov dx, 0x53a0            ; baa053
0001027c: (                    ): test byte ptr ds:0x211, 0x80 ; f606110280
00010281: (                    ): jz .+4                    ; 7404
00010283: (                    ): mov dx, word ptr ds:0x224 ; 8b162402
00010287: (                    ): add dx, 0x0400            ; 81c20004
0001028b: (                    ): jnb .+2                   ; 7302
0001028d: (                    ): xor dx, dx                ; 31d2
0001028f: (                    ): and dx, 0xfffc            ; 83e2fc
00010292: (                    ): jnz .+3                   ; 7503
00010294: (                    ): mov dx, 0xfffc            ; bafcff
00010297: (                    ): mov ss, ax                ; 8ed0
00010299: (                    ): movzx esp, dx             ; 660fb7e2
0001029d: (                    ): sti                       ; fb
```

此时赶紧打开arch/x86/boot/header.S确认一下反汇编的结果。

```
	# offset 512, entry point

	.globl	_start
_start:
		# Explicitly enter this as bytes, or the assembler
		# tries to generate a 3-byte jump here, which causes
		# everything else to push off to the wrong offset.
		.byte	0xeb		# short (2-byte) jump
		.byte	start_of_setup-1f
1:

    ...

	.section ".entrytext", "ax"
start_of_setup:
# Force %es = %ds
	movw	%ds, %ax
	movw	%ax, %es
	cld

# Apparently some ancient versions of LILO invoked the kernel with %ss != %ds,
# which happened to work by accident for the old code.  Recalculate the stack
# pointer if %ss is invalid.  Otherwise leave it alone, LOADLIN sets up the
# stack behind its own code, so we can't blindly put it directly past the heap.

	movw	%ss, %dx
	cmpw	%ax, %dx	# %ds == %ss?
	movw	%sp, %dx
	je	2f		# -> assume %sp is reasonably set

	# Invalid %ss, make up a new stack
	movw	$_end, %dx
	testb	$CAN_USE_HEAP, loadflags
	jz	1f
	movw	heap_end_ptr, %dx
1:	addw	$STACK_SIZE, %dx
	jnc	2f
	xorw	%dx, %dx	# Prevent wraparound

2:	# Now %dx should point to the end of our stack space
	andw	$~3, %dx	# dword align (might as well...)
	jnz	3f
	movw	$0xfffc, %dx	# Make sure we're not zero
3:	movw	%ax, %ss
	movzwl	%dx, %esp	# Clear upper half of %esp
	sti			# Now we should have a working stack
```

实模式内核512偏移处先是一个jmp，接下来20条指令和bochs中反汇编的是一模一样啊。这不就是咱要找的吗！

### 反汇编保护模式内核代码

已经看到了实模式内核的真容，那接下来就看看保护模式的内核吧。

```
<bochs:12> c
(0) Breakpoint 2, 0x0000000000100000 in ?? ()
Next at t=135456615
(0) [0x0000000000100000] 0010:0000000000100000 (unk. ctxt): cld                       ; fc
<bochs:13> creg
CR0=0x60000011: pg CD NW ac wp ne ET ts em mp PE
CR2=page fault laddr=0x0000000000000000
CR3=0x0000000000000000
    PCD=page-level cache disable=0
    PWT=page-level write-through=0
CR4=0x00000000: smep osxsave pcid fsgsbase smx vmx osxmmexcpt osfxsr pce pge mce pae pse de tsd pvi vme
CR8: 0x0
EFER=0x00000000: ffxsr nxe lma lme sce
```

我们直接continue后，就停在了0x100000的地址。这个就是我们刚才设置保护模式内核的断点地址。

查看寄存器，此时保护模式确实已经打开，CR0的PE是大写的。不过页表还没有开启。

```
<bochs:14> u /20
00100000: (                    ): cld                       ; fc
00100001: (                    ): cli                       ; fa
00100002: (                    ): lea esp, dword ptr ds:[esi+488] ; 8da6e8010000
00100008: (                    ): call .+0                  ; e800000000
0010000d: (                    ): pop ebp                   ; 5d
0010000e: (                    ): sub ebp, 0x0000000d       ; 83ed0d
00100011: (                    ): lea eax, dword ptr ss:[ebp+10657808] ; 8d8510a0a200
00100017: (                    ): mov dword ptr ds:[eax+2], eax ; 894002
0010001a: (                    ): lgdt ds:[eax]             ; 0f0110
0010001d: (                    ): mov eax, 0x00000018       ; b818000000
00100022: (                    ): mov ds, ax                ; 8ed8
00100024: (                    ): mov es, ax                ; 8ec0
00100026: (                    ): mov fs, ax                ; 8ee0
00100028: (                    ): mov gs, ax                ; 8ee8
0010002a: (                    ): mov ss, ax                ; 8ed0
0010002c: (                    ): lea esp, dword ptr ss:[ebp+10682368] ; 8da50000a300
00100032: (                    ): push 0x00000008           ; 6a08
00100034: (                    ): lea eax, dword ptr ss:[ebp+60] ; 8d853c000000
0010003a: (                    ): push eax                  ; 50
0010003b: (                    ): retf                      ; cb
```

反汇编一下代码，我们继续来看看是不是我们期待的。

这时候要看哪里的代码呢？对了，是arch/x86/boot/compressed/head_64.S。其中startup_32就是我们要找的。

```
	.code32
SYM_FUNC_START(startup_32)
	/*
	 * 32bit entry is 0 and it is ABI so immutable!
	 * If we come here directly from a bootloader,
	 * kernel(text+data+bss+brk) ramdisk, zero_page, command line
	 * all need to be under the 4G limit.
	 */
	cld
	cli

	leal	(BP_scratch+4)(%esi), %esp
	call	1f
1:	popl	%ebp
	subl	$ rva(1b), %ebp

	/* Load new GDT with the 64bit segments using 32bit descriptor */
	leal	rva(gdt)(%ebp), %eax
	movl	%eax, 2(%eax)
	lgdt	(%eax)

	/* Load segment registers with our descriptors */
	movl	$__BOOT_DS, %eax
	movl	%eax, %ds
	movl	%eax, %es
	movl	%eax, %fs
	movl	%eax, %gs
	movl	%eax, %ss

	/* Setup a stack and load CS from current GDT */
	leal	rva(boot_stack_end)(%ebp), %esp

	pushl	$__KERNEL32_CS
	leal	rva(1f)(%ebp), %eax
	pushl	%eax
	lretl
```

startup_32开始的20条指令和反汇编里显示的是不是也是一模一样？那就说明我们又找对啦。

至此，我们已经做好了用bochs探索内核进入start_kernel前的准备。感觉就像那黑暗的隧道里，照进了光。

PS： 感谢《自己动手写操作系统》，没有它我可能还在黑暗中摸索。

# 保护模式内核代码赏析

有了这么好的装备，干脆就研究一下保护模式下的部分代码。

## 计算当前内核被加载的地址

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

[1]: /brief_tutorial_on_kbuild/14_bzImage_whole_picture.md
[2]: https://tool.frogg.fr/Tutorial_Syslinux
[3]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/arch/x86/boot.rst?h=v6.7&id=0dd3ee31125508cd67f7e7172247f05b7fd1753a
[4]: https://github.com/RichardWeiYang/syslinux/tree/debug-kernel-load