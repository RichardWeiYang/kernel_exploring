了解完了内核加载过程后，我们来总结一下。将加载过程分为几个阶段来整理一下。

# bootloader加载bzImage

```
          +----------------------------+
          |                            |
          .                            .
          |                            |
          +----------------------------+
          |  vmlinux.bin               |
          |                            |
0x100000  +----------------------------+
          |                            |
          .                            .
          |                            |
          +----------------------------+
          |  setup.bin                 |
 0x10000  +----------------------------+
          |                            |
          .                            .
          |                            |
 0x00000  +----------------------------+
```

首先bootloader应该是按照boot protocol将内核加载到内存。bzImage其实分成两部分，setup.bin和vmlinux.bin。

* setup.bin是实模式内核
* vmlinux.bin是保护模式内核

具体可以看[bootloader如何加载bzImage][1]中最上面的那个图。

另外值的注意的一点是实模式内核虽然加载到了0x10000，但是跳入运行的是0x10200。因为前512字节是用来做boot sector的。

# 从实模式进入保护模式

setup.bin的使命就是让系统切换到保护模式。这部分的工作在protected_mode_jump完成。

内存布局没有变化，这里也就不再描述。

# 移动保护模式内核，并进入移动后的内核

[bootloader如何加载bzImage][1]我们看到vmlinux.bin有个神操作，就是把根目录的vmlinux给压缩后包进去了。所以我们想要运行真正的vmlinux，还得要先把vmlinux给解压出来。

不过在解压缩前，还要做点准备工作。因为内核用的是in place decompression，所以我们先要把这一大包东西移动到计算好的位置，再开始解压缩。

那移动到哪里呢？这个其实也是内核配置好的。在没有配置CONFIG_RELOCATABLE的时候，我们目标将把压缩的内核解压缩到CONFIG_PHYSICAL_START这个地址。这个地址的默认配置是0x1000000。因为要做in place decompression，所以会先把保护模式内核移动到0x1000000后面的某一个位置。这样解压缩后0x1000000开始的内容就是vmlinux了。


```
          +----------------------------+
          |                            |
          |  vmlinux.bin               |
          |                            |
          .                            .
          |                            |
0x1000000 +----------------------------+
          |                            |
          .                            .
          |                            |
          +----------------------------+
          |  vmlinux.bin               |
          |                            |
0x100000  +----------------------------+
          |                            |
          .                            .
          |                            |
          +----------------------------+
          |  setup.bin                 |
 0x10000  +----------------------------+
          |                            |
          .                            .
          |                            |
 0x00000  +----------------------------+
```

注意，此时内存0x100000开始的一段空间，只有靠近尾部的地方存放了vmlinux.bin。等解压缩后此处才有我们想要的内容。

# 解压缩压缩内核，并按照PHD加载

这部分的工作主要在decompress_kernel中完成，但其中经历了两个变化。

首先是将压缩内核就地解压， __decompress()。

```
          +----------------------------+
          |                            |
          .  vmlinux                   .
          |                            |
0x1000000 +----------------------------+
          |                            |
          .                            .
          |                            |
          +----------------------------+
          |  vmlinux.bin               |
          |                            |
0x100000  +----------------------------+
          |                            |
          .                            .
          |                            |
          +----------------------------+
          |  setup.bin                 |
 0x10000  +----------------------------+
          |                            |
          .                            .
          |                            |
 0x00000  +----------------------------+
```

这个时候，0x1000000起始的内存里，就存放了完整的vmlinux这个elf文件。这也是为什么parse_elf直接从output地址开始解析。

解压缩完后，就是第二步按照elf格式加载，parse_elf()。比如我这次编译出的vmlinux的加载信息如下：

```
readelf -l vmlinux

Elf file type is EXEC (Executable file)
Entry point 0x1000000
There are 5 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000200000 0xffffffff81000000 0x0000000001000000
                 0x00000000017e516c 0x00000000017e516c  R E    0x200000
  LOAD           0x0000000001a00000 0xffffffff82800000 0x0000000002800000
                 0x00000000003ef000 0x00000000003ef000  RW     0x200000
  LOAD           0x0000000001e00000 0x0000000000000000 0x0000000002bef000
                 0x00000000000320a8 0x00000000000320a8  RW     0x200000
  LOAD           0x0000000002022000 0xffffffff82c22000 0x0000000002c22000
                 0x00000000003d6000 0x0000000000606000  RWE    0x200000
  NOTE           0x00000000019e5118 0xffffffff827e5118 0x00000000027e5118
                 0x0000000000000054 0x0000000000000054         0x4
```

parse_elf会根据上面ProgramHeader的信息，将代码加载到指定位置。(PS: 没有配置CONFIG_RELOCATABLE的情况下)

```
          +----------------------------+
          |                            |
          .  vmlinux ELF loaded        .
          |                            |
0x1000000 +----------------------------+
          |                            |
          .                            .
          |                            |
          +----------------------------+
          |  vmlinux.bin               |
          |                            |
0x100000  +----------------------------+
          |                            |
          .                            .
          |                            |
          +----------------------------+
          |  setup.bin                 |
 0x10000  +----------------------------+
          |                            |
          .                            .
          |                            |
 0x00000  +----------------------------+
```

所以parse_elf后，0x1000000处的内容又有变化。当然，这个地址也就是我们真正进入vmlinux时的物理地址。

# 进入加载后内核，并按照虚拟地址建立页表，从物理地址切换到虚拟地址

刚才看到内核解压缩完成，也知道解压缩后vmlinux开始运行时的真正地址，基本也就对内核的加载了解清楚了。不过其实还差最后一步。

我们之前所有的操作都用的是物理地址，比如0x1000000。但是我们在运行中看到的内核地址都是0xfff开头的虚拟地址。这就涉及到内核页表的构建了。在[内核早期页表][2]，我们探索了内核是如何映射高地址的内核虚拟地址的。

这里我们就再来看一下，内核页表设置完后，是怎么切换到虚拟地址的。

```
	/*
	 * Switch to early_top_pgt which still has the identity mappings
	 * present.
	 */
	movq	%rax, %cr3

	/* Branch to the common startup code at its kernel virtual address */
	ANNOTATE_RETPOLINE_SAFE
	jmp	*0f(%rip)

	__INITRODATA
0:	.quad	common_startup_64

```

这个是更换页表后执行的代码，也就是跳转到0f处保存的地址common_startup_64。我们用nm来查看一下这个值。

```
nm vmlinux | grep common_startup_64
ffffffff81030dd8 t common_startup_64
```

用bochs设置断点0x1000000，然后一步步执行到更换cr3后，继续next。

```
<bochs:11> n
Next at t=617328373
(0) [0x0000000001030dd8] 0010:ffffffff81030dd8 (unk. ctxt): mov edx, 0x00001020       ; ba20100000
```

此时的地址是不是正好是common_startup_64的地址0xffffffff81030dd8。

然后我们再反汇编一下，确认接下来要执行的确实是common_startup_64的代码。

```
<bochs:12> u /10
WARNING: Offset 81030DD8 is out of selector 0010 limit (00000000...ffffffff)!
81030dd8: (                    ): mov edx, 0x00001020       ; ba20100000
81030ddd: (                    ): or edx, 0x00000040        ; 83ca40
81030de0: (                    ): mov rcx, cr4              ; 0f20e1
81030de3: (                    ): and ecx, edx              ; 21d1
81030de5: (                    ): bts ecx, 0x04             ; 0fbae904
81030de9: (                    ): mov cr4, rcx              ; 0f22e1
81030dec: (                    ): bts ecx, 0x07             ; 0fbae907
81030df0: (                    ): mov cr4, rcx              ; 0f22e1
81030df3: (                    ): mov ecx, dword ptr ds:[rip+25211399] ; 8b0d07b28001
81030df9: (                    ): test ecx, 0x80000000      ; f7c100000080
```

看着没有问题。

至此，整个内核bzImage加载的流程就梳理完了。

[1]: /load_kernel/02_how_bzImage_loaded.md
[2]: /kernel_pagetable/02-pagetable_compiled_in.md