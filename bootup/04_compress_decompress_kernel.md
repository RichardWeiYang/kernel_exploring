和应用程序一样，内核编译的时候是一个ELF文件，需要被加载到内存里，然后经过神奇一跃跳入内核执行。

内核是如何找到自己应该加载的物理地址的？又是如何在页表里建立虚拟地址映射的？这些都困扰着我。让我尝试看看是否能够解答。

# 从piggy.S开始

在[bzImage的全貌][1]中，我们看到内核是通过include的方式包含在了一个压缩内核中的。当时我们就是知道一个大概，这里需要再展开看看。

piggy.S的代码较短，我们贴上来看看。

```
.section ".rodata..compressed","a",@progbits
.globl z_input_len
z_input_len = 9993406
.globl z_output_len
z_output_len = 37640768
.globl input_data, input_data_end
input_data:
.incbin "arch/x86/boot/compressed/vmlinux.bin.zst"
input_data_end:


.section ".rodata","a",@progbits
.globl input_len
input_len:
	.long 9993406
.globl output_len
output_len:
	.long 37640768
```

其中定义了两个section，这两个section都能在arch/x86/boot/compressd/vmlinux.lds.S中找到对应的。
每个section里定义了点变量，就是这么简单。不过这次我们要看的是这个值是怎么来的。

具体的可以看代码mkpiggy.c，因为这个piggy.S是mkpiggy生成的。这里面的值是对应解压缩来说的：

* input是指解压缩的输入
* output是指解压缩的输出

所以我们看到input_len小于output_len。这两个值就应该是压缩后的文件大小和压缩前的文件大小。让我们来看看是不是

```
$ll vmlinux.bin*
-rwxrwxr-x 1 richard richard 37640768  3月 11 15:25 vmlinux.bin
-rw-rw-r-- 1 richard richard  9993406  3月 11 15:26 vmlinux.bin.zst
```

在arch/x86/boot/compressed目录下的这两个文件正是压缩前后的文件。其大小和piggy.S中生成的内容一致。

# 解压缩内核

内核跳转到保护模式后，设置了基本的段寄存器和页表后，就是要处理内核的解压缩了。这里我们就来看看整个解压缩的过程。

## 获得解压缩内核的起始地址

```
#ifdef CONFIG_RELOCATABLE
	leaq	startup_32(%rip) /* - $startup_32 */, %rbp
	movl	BP_kernel_alignment(%rsi), %eax
	decl	%eax
	addq	%rax, %rbp
	notq	%rax
	andq	%rax, %rbp
	cmpq	$LOAD_PHYSICAL_ADDR, %rbp
	jae	1f
#endif
	movq	$LOAD_PHYSICAL_ADDR, %rbp
```

这部分在上一篇笔记中已经详细看过，最后计算出的值是CONFIG_PHYSICAL_START配置的16M。而且这个值保存在了rbp中，备用。

## 移动压缩内核

```
/*
 * Copy the compressed kernel to the end of our buffer
 * where decompression in place becomes safe.
 */
	leaq	(_bss-8)(%rip), %rsi
	leaq	rva(_bss-8)(%rbx), %rdi
	movl	$(_bss - startup_32), %ecx
	shrl	$3, %ecx
	std
	rep	movsq
	cld
```

上面这段上一篇也看过了，就是把startup_32到_bss这一段代码都拷贝到了rbx为结尾的内存。为了避免覆盖，所以是从后面往前进行的拷贝。但是这个rbx是怎么来的呢？看看下面这段代码。

```
	/* Target address to relocate to for decompression */
	movl	BP_init_size(%rsi), %ebx
	subl	$ rva(_end), %ebx
	addq	%rbp, %rbx
```

上面的代码可以写成：
    rbx = rbp + init_size - (startup_32 - _end)
因为startup_32是0，
=>  rbx = rbp + init_size - _end

PS: startup_32的值是0，也可以通过nm命令来确认
```
$nm arch/x86/boot/compressed/vmlinux | grep startup_32
0000000000000000 T startup_32
```

接下来我们来看看BP_init_size(%rsi)是什么。rsi实际上是指向了boot_param，所以这个值是boot_param中init_size的保存的值。这个值在arch/x86/boot/header.S中定义。

```
init_size:		.long INIT_SIZE		# kernel initialization size
```

但是这个INIT_SIZE的定义有很多分岔情况，我这里只列出自己实验机器和配置上的情况。

```
#define ZO_z_extra_bytes	((ZO_z_output_len >> 8) + 131072)
# define ZO_z_extract_offset	(ZO_z_output_len + ZO_z_extra_bytes - \
				 ZO_z_input_len)
# define ZO_z_min_extract_offset ((ZO_z_extract_offset + 4095) & ~4095)
#define ZO_INIT_SIZE	(ZO__end - ZO_startup_32 + ZO_z_min_extract_offset)
# define INIT_SIZE ZO_INIT_SIZE
```

至于为什么这么定义，大家可以看上面的注释，我是没有仔细看，反正和解压缩有关。
这么一堆定义，我们整理一下

```
INIT_SIZE = (ZO__end - ZO_startup_32 + ZO_z_min_extract_offset)
          = (ZO__end - ZO_startup_32 + ((ZO_z_extract_offset + 4095) & ~4095))
          = (ZO__end - ZO_startup_32 +
                 (((ZO_z_output_len + ZO_z_extra_bytes - ZO_z_input_len) + 4095) & ~4095))
          = (ZO__end - ZO_startup_32 +
                 (((ZO_z_output_len + ((ZO_z_output_len >> 8) + 131072) - ZO_z_input_len) + 4095) & ~4095))
```

那这些ZO_xxx都是什么呢？这些是arch/x86/boot/zoffset.h中定义的。而这里面的值也是通过nm arch/x86/boot/compressed/vmlinux得到的，在原先的符号上加了ZO_作为定义名称。

如果你想验证一下是否如此，可以make arch/x86/boot/header.s，看一下预编译的文件内容是否符合。下面就是我实验时header.s的结果。

```
init_size: .long (0x0000000000a1e000 - 0x0000000000000000 + 
      (((0x00000000023e7a40 + ((0x00000000023e7a40 >> 8) + 131072) - 0x000000000099308d) + 4095) & ~4095)) # kernel initialization size
```

而zoffset.h中的值为（调整了下顺序，方便对比）

```
#define ZO__end 0x0000000000a1e000
#define ZO_startup_32 0x0000000000000000
#define ZO_z_output_len 0x00000000023e7a40
#define ZO_z_input_len 0x000000000099308d
```

看来完全符合～

总之，计算了一通后，我们终于在解压缩内核地址rbp的基础上增加了一个安全的偏移，得到了我们要移动压缩内核的地址rbx。

## 原地解压

内核玩的是 in-place decompression。估计是以前内存紧张，要好好计算该放到哪里才能不在解压缩的时候破坏内存现场。这也是为什么刚才的INIT_SIZE计算得这么辛苦。

```
/*
 * Do the extraction, and jump to the new kernel..
 */
	/* pass struct boot_params pointer and output target address */
	movq	%r15, %rdi
	movq	%rbp, %rsi
	call	extract_kernel		/* returns kernel entry point in %rax */

/*
 * Jump to the decompressed kernel.
 */
	movq	%r15, %rsi
	jmp	*%rax
```

extract_kernel一共有两个参数，boot_param和目标地址。也就是我们刚才算出来的rbp。
返回也就一个参数entry_point，保存在rax中。

我们来看看究竟会返回什么样的地址.(没有高级功能的情况下)

```
extract_kernel(void *rmode, unsigned char *output)
    entry_offset = decompress_kernel(output, virt_addr, error);
        __decompress(..., output, ...);
        entry = parse_elf(output);
            return ehdr.e_entry - LOAD_PHYSICAL_ADDR;
        return entry;
    return output + entry_offset;
```

首先将压缩内核解压，到output开始的内存中。也就是我们刚才计算得到的rbp=16M的地址。
然后parse_elf。注意，这个时候，output的内容已经是内核根目录下的vmlinux，而不是arch/x86/boot下的任何一个vmlinux了。所以这里的ehdr.e_entry是根目录下vmlinux的入口地址.

因为output=rbp，在这次计算中就是LOAD_PHYSICAL_ADDR。现在我们展开一下extract_kernel的返回值看看：

```
output + entry_offset = output + ehdr.e_entry - LOAD_PHYSICAL_ADDR
                      = LOAD_PHYSICAL_ADDR + ehdr.e_entry - LOAD_PHYSICAL_ADDR
                      = ehdr.e_entry
```

也就是说，我们预期解压缩后，得到的跳转地址就是ehdr.e_entry。那我们现在看看vmlinux中的entry值，再用bochs来验证一下。

```
$readelf -h vmlinux
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x1000000
  Start of program headers:          64 (bytes into file)
...
```

我们看到Entry point address: 0x1000000。先看一下extract_kernel返回后的寄存器状态。

```
<bochs:43> u /2
03429fb2: (                    ): mov rsi, r15              ; 4c89fe
03429fb5: (                    ): jmp rax                   ; ffe0
<bochs:44> r
rax: 0x00000000_01000000 rcx: 0x00000000_00000000
rdx: 0x00000000_000003d5 rbx: 0x00000000_02aa2000
```

确实是0x1000000，如我们所预料的。我们跳进取，反汇编一下看看。

```
<bochs:48> u /6
01000000: (                    ): mov r15, rsi              ; 4989f7
01000003: (                    ): lea rsp, qword ptr ds:[rip+25182030] ; 488d254e3f8001
0100000a: (                    ): lea rdi, qword ptr ds:[rip-17] ; 488d3defffffff
01000011: (                    ): mov ecx, 0xc0000101       ; b9010100c0
01000016: (                    ): lea rdx, qword ptr ds:[rip+29265891] ; 488d15e38fbe01
0100001d: (                    ): mov eax, edx              ; 89d0
```

正好是startup_64里的代码。好了，终于来到了我们熟知的内核了！

慢着，为啥是startup_64呢？这还要从链接和加载说起。

[1]: /brief_tutorial_on_kbuild/14_bzImage_whole_picture.md