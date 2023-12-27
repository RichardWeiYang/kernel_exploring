
# 看看这个build都做了什么～
还好这是个用户态的c语言，咱看得懂～

## boot_flag 0xAA55
在几乎最开始的地方就来了这么一句
```
	if (get_unaligned_le16(&buf[510]) != 0xAA55)
		die("Boot block hasn't got boot flag (0xAA55)");
```
这是读取setup.bin文件510偏移的地方。

setup.bin是setup.elf的objcopy, setup.elf是一坨obj文件链接的。。。

不过结合了grep和setup.ld文件，发现了些东西。

```
header.S中

	.section ".header", "a"
	.globl	sentinel
sentinel:	.byte 0xff, 0xff        /* Used to detect broken loaders */

	.globl	hdr
hdr:
setup_sects:	.byte 0			/* Filled in by build.c */
root_flags:	.word ROOT_RDONLY
syssize:	.long 0			/* Filled in by build.c */
ram_size:	.word 0			/* Obsolete */
vid_mode:	.word SVGA_MODE
root_dev:	.word 0			/* Filled in by build.c */
boot_flag:	.word 0xAA55
```
从.header这个section到boot_flag有15个字节。

```
再看一下setup.ld

	. = 495;
	.header		: { *(.header) }
```

而495 + 15 = 510，哦～ 好巧。

为了证实这个发现，在header.S处改成0xAA56， 在build.c中打印取出来的值。make bzImage显示“AA 56”。证明了我的发现是对的~

好了，虽然没干什么正事，有这么个发现，还是挺开心的～

## pe_header & nr_sections
通过上一节学到的知识，又找到两个有意思的。

build.c中有
```
	pe_header = get_unaligned_le32(&buf[0x3c]);
	num_sections = get_unaligned_le16(&buf[pe_header + 6]);
```
这里的buf还是setup.bin读过来的。

然后依样画葫芦在header.S中找
```
#ifdef CONFIG_EFI_STUB
	.org	0x3c
	#
	# Offset to the PE header.
	#
	.long	pe_header
#endif /* CONFIG_EFI_STUB */

	.section ".bsdata", "a"
bugger_off_msg:
	.ascii	"Use a boot loader.\r\n"
	.ascii	"\n"
	.ascii	"Remove disk and press any key to reboot...\r\n"
	.byte	0

#ifdef CONFIG_EFI_STUB
pe_header:
	.ascii	"PE"
	.word 	0

coff_header:
#ifdef CONFIG_X86_32
	.word	0x14c				# i386
#else
	.word	0x8664				# x86-64
#endif
	.word	4				# nr_sections
```

我只能说人注释写得真好。

开始我有点疑惑，那这个.org表示一个offset（没查到正规语法，自己猜是这个作用，有知道的大侠请指教。）会不会和section指定的冲突？

转过来又看了一眼setup.ld
```
	. = 0;
	.bstext		: { *(.bstext) }
	.bsdata		: { *(.bsdata) }

```
恩，那我猜，人作者是算好了的～

赶车上班了！

## update_pecoff_section_header_fields()

在build.c中有这么个函数，它会在这个地方被间接调用到。
```
static void update_pecoff_setup_and_reloc(unsigned int size)
{
	u32 setup_offset = 0x200;
	u32 reloc_offset = size - PECOFF_RELOC_RESERVE;
	u32 setup_size = reloc_offset - setup_offset;

	update_pecoff_section_header(".setup", setup_offset, setup_size);
```

看不懂这是在干啥，先看看这个函数做了点啥吧。
```
#ifdef CONFIG_X86_32
	section = &buf[pe_header + 0xa8];
#else
	section = &buf[pe_header + 0xb8];
#endif

	while (num_sections > 0) {
		if (strncmp((char*)section, section_name, 8) == 0) {
			/* section header size field */
			put_unaligned_le32(size, section + 0x8);

			/* section header vma field */
			put_unaligned_le32(vma, section + 0xc);

			/* section header 'size of initialised data' field */
			put_unaligned_le32(datasz, section + 0x10);

			/* section header 'file offset' field */
			put_unaligned_le32(offset, section + 0x14);

			break;
		}
		section += 0x28;
		num_sections--;
	}
```

接着上一个小节的结尾，pe_header实际上是header.S中的某个位置。那pe_header + 0xa8是指向哪里？搜了一下header.S，没有。。。 那数吧。。。略长，慎入

```
#ifdef CONFIG_EFI_STUB
pe_header:
	0 .ascii	"PE"
	2 .word 	0

coff_header:
#ifdef CONFIG_X86_32
	4 .word	0x14c				# i386
#else
	.word	0x8664				# x86-64
#endif
	6 .word	4				# nr_sections
	8 .long	0 				# TimeDateStamp
	12 .long	0				# PointerToSymbolTable
	16 .long	1				# NumberOfSymbols
	20 .word	section_table - optional_header	# SizeOfOptionalHeader
#ifdef CONFIG_X86_32
	22 .word	0x306				# Characteristics.
						# IMAGE_FILE_32BIT_MACHINE |
						# IMAGE_FILE_DEBUG_STRIPPED |
						# IMAGE_FILE_EXECUTABLE_IMAGE |
						# IMAGE_FILE_LINE_NUMS_STRIPPED
#else
	.word	0x206				# Characteristics
						# IMAGE_FILE_DEBUG_STRIPPED |
						# IMAGE_FILE_EXECUTABLE_IMAGE |
						# IMAGE_FILE_LINE_NUMS_STRIPPED
#endif

optional_header:
#ifdef CONFIG_X86_32
	24 .word	0x10b				# PE32 format
#else
	.word	0x20b 				# PE32+ format
#endif
	26 .byte	0x02				# MajorLinkerVersion
	27 .byte	0x14				# MinorLinkerVersion

	# Filled in by build.c
	28 .long	0				# SizeOfCode

	32 .long	0				# SizeOfInitializedData
	36 .long	0				# SizeOfUninitializedData

	# Filled in by build.c
	40 .long	0x0000				# AddressOfEntryPoint

	44 .long	0x0200				# BaseOfCode
#ifdef CONFIG_X86_32
	48 .long	0				# data
#endif

extra_header_fields:
#ifdef CONFIG_X86_32
	52 .long	0				# ImageBase
#else
	.quad	0				# ImageBase
#endif
	56 .long	0x20				# SectionAlignment
	60 .long	0x20				# FileAlignment
	64 .word	0				# MajorOperatingSystemVersion
	66 .word	0				# MinorOperatingSystemVersion
	68 .word	0				# MajorImageVersion
	70 .word	0				# MinorImageVersion
	72 .word	0				# MajorSubsystemVersion
	74 .word	0				# MinorSubsystemVersion
	76 .long	0				# Win32VersionValue

	#
	# The size of the bzImage is written in tools/build.c
	#
	80 .long	0				# SizeOfImage

	84 .long	0x200				# SizeOfHeaders
	88 .long	0				# CheckSum
	92 .word	0xa				# Subsystem (EFI application)
	94 .word	0				# DllCharacteristics
#ifdef CONFIG_X86_32
	96 .long	0				# SizeOfStackReserve
	100 .long	0				# SizeOfStackCommit
	104 .long	0				# SizeOfHeapReserve
	108 .long	0				# SizeOfHeapCommit
#else
	.quad	0				# SizeOfStackReserve
	.quad	0				# SizeOfStackCommit
	.quad	0				# SizeOfHeapReserve
	.quad	0				# SizeOfHeapCommit
#endif
	112 .long	0				# LoaderFlags
	116 .long	0x6				# NumberOfRvaAndSizes

	120 .quad	0				# ExportTable
	128 .quad	0				# ImportTable
	136 .quad	0				# ResourceTable
	144 .quad	0				# ExceptionTable
	152 .quad	0				# CertificationTable
	160 .quad	0				# BaseRelocationTable

	# Section table
section_table:
	#
	# The offset & size fields are filled in by build.c.
	#
	168 .ascii	".setup"
```

眼神不好，一共数了三遍终于数对了。。。 数完了之后，发现了点海阔天空的东西。

```
	# Section table
section_table:
	#
	# The offset & size fields are filled in by build.c.
	#
	.ascii	".setup"
	.byte	0
	.byte	0
	.long	0
	.long	0x0				# startup_{32,64}
	.long	0				# Size of initialized data
						# on disk
	.long	0x0				# startup_{32,64}
	.long	0				# PointerToRelocations
	.long	0				# PointerToLineNumbers
	.word	0				# NumberOfRelocations
	.word	0				# NumberOfLineNumbers
	.long	0x60500020			# Characteristics (section flags)

	#
	# The EFI application loader requires a relocation section
	# because EFI applications must be relocatable. The .reloc
	# offset & size fields are filled in by build.c.
	#
	.ascii	".reloc"
	.byte	0
	.byte	0
	.long	0
	.long	0
	.long	0				# SizeOfRawData
	.long	0				# PointerToRawData
	.long	0				# PointerToRelocations
	.long	0				# PointerToLineNumbers
	.word	0				# NumberOfRelocations
	.word	0				# NumberOfLineNumbers
	.long	0x42100040			# Characteristics (section flags)

	#
	# The offset & size fields are filled in by build.c.
	#
	.ascii	".text"
	.byte	0
	.byte	0
	.byte	0
	.long	0
	.long	0x0				# startup_{32,64}
	.long	0				# Size of initialized data
						# on disk
	.long	0x0				# startup_{32,64}
	.long	0				# PointerToRelocations
	.long	0				# PointerToLineNumbers
	.word	0				# NumberOfRelocations
	.word	0				# NumberOfLineNumbers
	.long	0x60500020			# Characteristics (section flags)

	#
	# The offset & size fields are filled in by build.c.
	#
	.ascii	".bss"
	.byte	0
	.byte	0
	.byte	0
	.byte	0
	.long	0
	.long	0x0
	.long	0				# Size of initialized data
						# on disk
	.long	0x0
	.long	0				# PointerToRelocations
	.long	0				# PointerToLineNumbers
	.word	0				# NumberOfRelocations
	.word	0				# NumberOfLineNumbers
	.long	0xc8000080			# Characteristics (section flags)
```

原来后面跟着四个"section"，这个section又不是链接概念的section。这下终于知道那个num_sections为啥定义成4了。

好了，这个函数就是根据编译链接晚的结果，填写这四个section结构体对应的域。具体有啥作用，后面再看～

PS：话说这汇编写得实在难看，数起来太累。。。

## update_pecoff_setup_and_reloc
再回过头来看看这个函数

```
static void update_pecoff_setup_and_reloc(unsigned int size)
{
	u32 setup_offset = 0x200;
	u32 reloc_offset = size - PECOFF_RELOC_RESERVE;
	u32 setup_size = reloc_offset - setup_offset;

	update_pecoff_section_header(".setup", setup_offset, setup_size);
	update_pecoff_section_header(".reloc", reloc_offset, PECOFF_RELOC_RESERVE);

	/*
	 * Modify .reloc section contents with a single entry. The
	 * relocation is applied to offset 10 of the relocation section.
	 */
	put_unaligned_le32(reloc_offset + 10, &buf[reloc_offset]);
	put_unaligned_le32(10, &buf[reloc_offset + 4]);
}
```
暂时还没有看懂，不过我猜测一下。

这个setup_offset设为0x200，转换成十进制是512。而在header.S中有这么一行。

```
	# offset 512, entry point

	.globl	_start
_start:
```
我觉得我是对的～ 不过再看一眼这个section中的注释

```
	.ascii	".setup"
	.byte	0
	.byte	0
	.long	0
	.long	0x0				# startup_{32,64}
	.long	0				# Size of initialized data
						# on disk
	.long	0x0				# startup_{32,64}
	.long	0				# PointerToRelocations
	.long	0				# PointerToLineNumbers
	.word	0				# NumberOfRelocations
	.word	0				# NumberOfLineNumbers
	.long	0x60500020			# Characteristics (section flags)
```
人在注释中写的这个是 startup_{32,64}的地址，好像又怎么对。
不着急，等以后看到了哪里用，再过来看看。

## 一不小心到了最后
虽然还是有没有看明白的地方，但是这个build就到头了。看一眼最后他做了点什么。
```
	crc = partial_crc32(buf, i, crc);
	if (fwrite(buf, 1, i, dest) != i)
		die("Writing setup failed");

	/* Copy the kernel code */
	crc = partial_crc32(kernel, sz, crc);
	if (fwrite(kernel, 1, sz, dest) != sz)
		die("Writing kernel failed");
```

简单明了。

最后总结一下这个build的过程。

bzImage由两个obj文件拼接而成，setup.bin vmlinux.bin。
在拼接的过程中，根据每次编译的结果，更新（动态填写）文件中的某些变量。

bzImage的编译过程就告一段落，然而心中的疑惑更多了。作为保存在磁盘上的一个文件，grub又是如何加载？加载到内存的哪个位置？加载时首先执行的是哪条指令？

路漫漫其修远兮，吾将上下而求索

# bzImage相关的文件及其关系

这是后加的，我觉得加到这里更加合适些
```

bzImage
  |                                                             
  +-- arch/x86/boot/setup.bin                                   
  |     |                                                       
  |     | objcopy from :                                        
  |     +-arch/x86/boot/setup.elf                               
  |                                                             
  |                                                             
  +-- arch/x86/boot/vmlinux.bin                                 
        |                                                       
        | objcopy from :                                        
        +-arch/x86/boot/compressed/vmlinux                      
            |                                                   
            | linked from :                                     
            +-arch/x86/boot/compressed/head_$(BITS).o           
            |                                                   
            +-arch/x86/boot/compressed/misc.o                   
            |                                                   
            +-arch/x86/boot/compressed/string.o                 
            |                                                   
            +-arch/x86/boot/compressed/cmdline.o                
            |                                                   
            +-arch/x86/boot/compressed/error.o                  
            |                                                   
            +-arch/x86/boot/compressed/cpuflags.o               
            |                                                   
            +-arch/x86/boot/compressed/piggy.o                  
                |                                               
                | linked with :                                 
                +-arch/x86/boot/compressed/vmlinux.bin.gz       
                    |                                           
                    |  compressed from :                        
                    +- vmlinux                                  
                                                                
```

[1]: http://blog.csdn.net/richardysteven/article/details/56482735
