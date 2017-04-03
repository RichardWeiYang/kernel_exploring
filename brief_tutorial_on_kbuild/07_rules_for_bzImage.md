# 原来安装的是它

编译完内核，接下来要做的就是安装新的内核来使用了。那安装的是哪个文件呢？

关于这个问题，昨天我还真好好看了一下，发现原来安装的不是根目录下的vmlinux，而是bzImage。玩了内核这么多年，一直都以为是根目录下的那个vmlinux是安装时拷贝到/boot/目录下的文件，结果原来不是。真是惭愧惭愧。

我是从make install这个规则开始找下去的。

```
boot := arch/x86/boot
install:
	$(Q)$(MAKE) $(build)=$(boot) $@
```

这个规则在arch/x86/Makefile中，好奇为什么没有在根Makefile里找到。

那实际上真正执行的是在arch/x86/boot/Makefile中这个规则

```
install:
	sh $(srctree)/$(src)/install.sh $(KERNELRELEASE) $(obj)/bzImage \
		System.map "$(INSTALL_PATH)"
```

对应x86架构，在这个install.sh就是arch/x86/boot/install.sh。虽然脚本中有几种安装内核的方式，不过我们只看其中一种也就能确认安装在/boot/目录下的是bzImage而不是vmlinux了。

```
cat $2 > $4/vmlinuz
```

所以说不看不知道，一看吓一跳。以后不敢说自己懂内核了。

#目标在哪里？

我们在[内核编译的小目标][1]一文中也提到过，bzImage是x86平台下默认的目标之一。但是并没有在根目录的Makefile中发现bzImage目标。 而在根目录的Makefile中的前面部分有

```
include $(srctree)/arch/$(SRCARCH)/Makefile
```

这个是不同的arch会include不同的文件，比如是x86的架构就会include arch/x86/Makefile

打开一看，果不其然

```
# Default kernel to build
all: bzImage
```

好了，我们终于找到这个bzImage的target了，在x86平台它也是all的一部分。

来看看具体是什么


# 生成规则

在arch/x86/Makefile中，bzImage具体定义是这么个样子的。

```
# KBUILD_IMAGE specify target image being built
KBUILD_IMAGE := $(boot)/bzImage

bzImage: vmlinux
ifeq ($(CONFIG_X86_DECODER_SELFTEST),y)
	$(Q)$(MAKE) $(build)=arch/x86/tools posttest
endif
	$(Q)$(MAKE) $(build)=$(boot) $(KBUILD_IMAGE)
	$(Q)mkdir -p $(objtree)/arch/$(UTS_MACHINE)/boot
	$(Q)ln -fsn ../../x86/boot/bzImage $(objtree)/arch/$(UTS_MACHINE)/boot/$@
```

又是一坨这么长的，整的好生心烦。幸好，看到了一个眼熟的\$(MAKE) \$(build)=\$(boot) \$(KBUILD_IMAGE)。这个不是我们编译具体目标的时候经常看到的么？咱来展开看一眼：

> make -f scripts/Makefile.build obj=arch/x86/boot arch/x86/boot/bzImage

是不是觉得好像亲切了一些？

对scripts/Makefile.build文件再补充一点，在该文件的开头处出了包含了scripts/Kbuild.include文件，还包含了一个$(build-file)文件。

先来看一下这个变量的定义：

```
#The filename Kbuild has precedence over Makefile
kbuild-dir := $(if $(filter /%,$(src)),$(src),$(srctree)/$(src))
kbuild-file := $(if $(wildcard $(kbuild-dir)/Kbuild),$(kbuild-dir)/Kbuild,$(kbuild-dir)/Makefile)
include $(kbuild-file)
```

你猜到了什么不？ 没猜到？ 再看一眼上面的注释？

对了，这个就是单个目录下符合kbuild系统的规则文件。

你还记得编译一个内核模块时候那个Makefile中定义的obj-y, obj-m么？为什么我们在内核模块中的规则文件只需要定义这几个变量，就可以编译出目标文件和模块呢？原因就是在scripts/Makefile.build中包含了目标目录下的规则文件。

不懂也没关系，看多了日后自然知晓。

先看一下当前包含的文件，这次包含的是arch/x86/boot/Makefile。里面有这么一句。

```
quiet_cmd_image = BUILD   $@
cmd_image = $(obj)/tools/build $(obj)/setup.bin $(obj)/vmlinux.bin \
			       $(obj)/zoffset.h $@

$(obj)/bzImage: $(obj)/setup.bin $(obj)/vmlinux.bin $(obj)/tools/build FORCE
    $(call if_changed,image)
    @echo 'Kernel: $@ is ready' ' (#'`cat .version`')'
```

简单明了，会心一笑。

所以最后的最后是通过tools/build这个用户态工具生成bzImage文件，而依赖的文件是setup.bin和vmlinux.bin。

当然加上绝对路径后，这几个文件分别是，

```
arch/x86/boot/tools/build
arch/x86/boot/setup.bin
arch/x86/boot/vmlinux.bin
arch/x86/boot/zoffset.h arch/x86/boot/bzImage                                                                              
```

# 看看这个build都做了什么～

这是一个用户态的程序，文件在arch/x86/boot/tools/build.c。虽然说main函数的主体结构相对简洁清晰，但实际上包含了不少文件格式相关的检查和动态填充。不少细节我也不是很懂，而且本主题关注的还是bzImage编译过程，所以细节部分就暂且略过。

来看我们能看得明白的，按照相关线索抽取了代码

回忆一下最后build的命令：

```
$(obj)/tools/build $(obj)/setup.bin $(obj)/vmlinux.bin \
			       $(obj)/zoffset.h $@
```

## 拷贝setup.bin

```
	dest = fopen(argv[4], "w");

	file = fopen(argv[1], "r");
	c = fread(buf, 1, sizeof(buf), file);

	if (fwrite(buf, 1, i, dest) != i)
		die("Writing setup failed");
```

* 先打开了bzImage，叫dest。
* 然后读取setup.bin到buf
* 最后写入bzImage

## 拷贝vmlinux.bin

```
	fd = open(argv[2], O_RDONLY);

	kernel = mmap(NULL, sz, PROT_READ, MAP_SHARED, fd, 0);

	/* Copy the kernel code */
	crc = partial_crc32(kernel, sz, crc);
	if (fwrite(kernel, 1, sz, dest) != sz)
		die("Writing kernel failed");
```

* 先打开了vmlinux.bin
* 做了一下map
* 拷贝到了dest指向的bzImage

是不是也挺简单的呢。

## 一张图显示依赖关系

```
         setup.bin    vmlinux.bin  
               \         /
                \       /
                 bzImage
```

丑是丑了点，毕竟清晰了些。

# 心中的疑惑

bzImage的编译过程就告一段落，然而心中的疑惑更多了。作为保存在磁盘上的一个文件，grub又是如何加载？加载到内存的哪个位置？加载时首先执行的是哪条指令？

bzImage的两个组成部分vmlinux.bin和setup.bin又是怎么出现的呢？

路漫漫其修远兮，吾将上下而求索


[1]: http://blog.csdn.net/richardysteven/article/details/56482735
