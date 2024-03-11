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

[1]: /brief_tutorial_on_kbuild/14_bzImage_whole_picture.md