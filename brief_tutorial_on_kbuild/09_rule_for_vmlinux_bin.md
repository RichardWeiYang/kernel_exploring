真没有想到，在编译过内核的源码目录下，你可以找到两个同叫vmlinux的文件。

```
$find . -name vmlinux
./vmlinux
./arch/x86/boot/compressed/vmlinux
```

怎么样，你之前有发现过么？这还是在探索vmlinux.bin的过程中发现的秘密。

他们究竟都是干什么用的？有什么联系？和bzImage之间有关联么？让我们来揭开这神秘的面纱。


# 隐藏的vmlinux

老规矩，先来看看vmlinux.bin的规则。

```
$(obj)/vmlinux.bin: $(obj)/compressed/vmlinux FORCE
	$(call if_changed,objcopy)
```

嗯，看到刚才find中发现的vmlinux了不？原来vmlinux.bin是这个vmlinux通过objcopy而来。那这个都包含了谁？ 又和根目录下的vmlinux有什么关系呢？你们俩真是太像了。

```
$(obj)/compressed/vmlinux: FORCE
	$(Q)$(MAKE) $(build)=$(obj)/compressed $@
```

原来老人家还有一个单独的目录，再次调用了make命令。

# 揭开面纱

检验基本功的时候又来了，还记得这个命令究竟是做了什么么？还记得这个时候，规则文件是要去哪里找么？如果想不起来可以去回顾前面几篇入门文章。

在arch/x86/boot/compressd/Makefile中，找到了规则：

```
$(obj)/vmlinux: $(vmlinux-objs-y) FORCE
	$(call if_changed,check_data_rel)
	$(call if_changed,ld)
```

偷个懒，猜一下，这个vmlinux就是把变量vmlinux-objs-y中的所有目标链接而成的。是不是觉得易如反掌了？

好，那来看看这个变量里都有谁。

```
vmlinux-objs-y := $(obj)/vmlinux.lds $(obj)/head_$(BITS).o $(obj)/misc.o \
	$(obj)/string.o $(obj)/cmdline.o $(obj)/error.o \
	$(obj)/piggy.o $(obj)/cpuflags.o
```

好少啊，感觉比setup.bin还少。貌似挺简单的啊，这么着就算是把整个bzImage的编译流程都走完了～

真的完了么？ 你有没有发现什么不对劲？

对了，我们的根目录的vmlinux呢？难道放到启动目录下的内核里面没有根目录的vmlinux？不对啊，根目录的vmlinux可是包含了所有内核真正的代码的啊。

是的，你发现的没错，我们还没有真正走到内核编译的最深处。

# 石破惊天

也不知道是什么机缘巧合，竟然发现在arch/x86/boot/compress/目录下面的piggy.S这个文件是编译时生成的。

在我的环境上，这个文件看上去像是这样：

```
.section ".rodata..compressed","a",@progbits
.globl z_input_len
z_input_len = 6957106
.globl z_output_len
z_output_len = 22677744
.globl input_data, input_data_end
input_data:
.incbin "arch/x86/boot/compressed/vmlinux.bin.gz"
input_data_end:
```

怎么样，你有没有觉得大跌眼镜？简直就是FXXK。

在汇编代码中直接包含了一个文件。让我们看看这个incbin语句是什么含义吧。

```
You can use INCBIN to include executable files, literals, or any arbitrary data. The contents of the file are added to the current ELF section, byte for byte, without being interpreted in any way.
```

哥们直接把整个内核给包进来了，小生佩服。

# 再烧一次脑

为了确认，我们再来看看这个vmlinux.bin.gz是不是真的是内核的代码。

注意，下面的关系还真是有点烧脑。

先来看vmlinux.bin.gz:

```
vmlinux.bin.all-y := $(obj)/vmlinux.bin

$(obj)/vmlinux.bin.gz: $(vmlinux.bin.all-y) FORCE
	$(call if_changed,gzip)
```

那再来看$(obj)/vmlinux.bin。注意哦，这个已经是arch/x86/boot/compressed/vmlinux.bin，而不是arxh/x86/boot/vmlinux.bin了。怎么样，是不是有点烧脑？

```
$(obj)/vmlinux.bin: vmlinux FORCE
	$(call if_changed,objcopy)
```

而这个时候的依赖，vmlinux，就是根目录下的vmlinux了。

At last。

终于，我们看清楚了内核代码是怎么样打包到bzImage中了。

# 一张图解说

我知道，你肯定已经晕的不能再晕了。我们还是用一张图来解说一下。

```
       vmlinux

         ||
         \/

       boot/compressed/vmlinux.bin


         ||
         \/

       boot/compressed/vmlinux.bin.gz

         ||
         \/

       piggy.o       head_$(BITS).o
                     misc.o
                     cmdline.o
                     error.o
                     cpuflsgs.o

             ||
             \/

      boot/compressed/vmlinux

             ||
             \/

         boot/vmlinux.bin
```

是不是稍微的清晰了那么一些些？

# 真相大白

终于知道了内核还会被压缩，还会被放在一个叫piggy.o的文件中，然后再被打包成一个bzImage放到启动目录中被加载。

没想到内核的大侠们也挺逗的，起个名字尽然还有点冷幽默。

整个内核的编译过程已经基本走完了，你可能会问知道这个编译的过程除了好玩还能有什么用呢？

我想除了能用来参加面试之外，可能还有一个原因，那就是你有机会明白内核启动时的页表是长什么样的。嗯？你不知道？不着急，你的内核之旅可能才刚刚开始。
