这篇文章是内核编译探索的重要知识点汇总，类似一个工具，列举了我所见到的一些target, rule. 以此希望对内核编译更进一步了解。

然而如果没有实际内核编译的经验，可能会觉得比较枯燥。建议从这个[内核编译探索系列][1]开始阅读。

# <font color=#8b0000>稍微谈一点架构</font>

文章写到一大半，发现可以来谈谈“架构”了。把这部分放到开始，或许会对大家理解有点帮助。

整个内核代码从make的角度来看，可能是这样的。

     +--  Makefile
     |
     +--+ init     
     |  |   
     |  +-- Makefile
     |      
     +--+ fs   
     |  |   
     |  +-- Makefile
     |  |   
     |  +-+ ext4
     |    | 
     |    +-- Makefile 
     |      
     +--+ kernel   
     |  |   
     |  +-- Makefile
     |  |   
     |  +-+ sched
     |    | 
     |    +-- Makefile 
     |      
     +--    

其实说白了就是每个层次目录中的Makefile中定义好相应的目标，如果有下一层，则进入到下一层继续编译。而这个过程中比较重要的文件是Makefile.build.

[更正]： “进入下一层”的表达有误。 在整个内核编译过程中，当前目录都是内核源码树的根目录。对不同目标，不同路径下的目标的编译，是通过obj变量做到的。这一点需要细看Makefile.build文件。

据说make中多线程同时编译，就是要依靠这么进入下一层来实现的。

# <font color=#8b0000>rules</font>
## <font color=#8b0000>if_changed_xxx</font>
这个规则实在是太重要，曝光率也是超高。

一般用法是这样的
```
$(obj)/bzImage: $(obj)/setup.bin $(obj)/vmlinux.bin $(obj)/tools/build FORCE
	$(call if_changed,image)
	@echo 'Kernel: $@ is ready' ' (#'`cat .version`')'
```

那来展开一下该定义在scripts/Kbuild.include中

```
###
# if_changed      - execute command if any prerequisite is newer than
#                   target, or command line has changed
# if_changed_dep  - as if_changed, but uses fixdep to reveal dependencies
#                   including used config symbols
# if_changed_rule - as if_changed but execute rule instead
# See Documentation/kbuild/makefiles.txt for more info
```
恩，原来有兄弟三个呢。先来看看大哥的样子
```
# Execute command if command has changed or prerequisite(s) are updated.
#
if_changed = $(if $(strip $(any-prereq) $(arg-check)),                       \
	@set -e;                                                             \
	$(echo-cmd) $(cmd_$(1));                                             \
	printf '%s\n' 'cmd_$@ := $(make-cmd)' > $(dot-target).cmd, @:)
```
第一眼看这么长，其实我的内心是有点抵触的。一层层来看吧

首先这是个if语句， 当条件为真，则执行一个，若为假，则执行另一个。这么来看，逐个击破～

###<font color=#8b0000>判断条件</font> 
```
$(strip $(any-prereq) $(arg-check))
```
意思就是所有依赖条件和arg-check字符串中如果有字符串，那么就会执行相关命令。
####<font color=#8b0000>有没有新的依赖</font>
```
# Find any prerequisites that is newer than target or that does not exist.
# PHONY targets skipped in both cases.
any-prereq = $(filter-out $(PHONY),$?) $(filter-out $(PHONY) $(wildcard $^),$^)
```
人注释写得真到位，一个表示比目标更新的依赖，一个表示还不存在的依赖。
先看看两个自动变量，

 -  $?     The names of all the prerequisites that are newer than the target
 -  $^    The names of all the prerequisites

第一个太明显，第二个要看一下wildcard函数的定义了，

> This string, used anywhere in a makefile, is replaced by a
> space-separated list of names of existing files that match one of the
> given file name patterns

恩，就是用来找到现在已经有的文件的。那结合定义，就是先找到依赖中已经存在的文件，然后从所有依赖中去掉。那不就剩下还没有生成的依赖了么。

####<font color=#8b0000>有没有arg-check变化</font>

```
# Check if both arguments are the same including their order. Result is empty
# string if equal. User may override this check using make KBUILD_NOCMDDEP=1
arg-check = $(filter-out $(subst $(space),$(space_escape),$(strip $(cmd_$@))), \
                         $(subst $(space),$(space_escape),$(strip $(cmd_$1))))
```
还真么怎么懂，除了注释。。。

总体上看，就是比较了下两个字符串，去掉一样的。那到底是哪两个字符串呢？
折腾了一下，终于有点眉目了。 

cmd_$1 这个$1是 call_changed函数的第一个参数，在这个例子里面展开后就是 cmd_image, 这个在arch/x86/boot/Makefile中定义。

那cmd_$@呢？ 按照逻辑展开就是 cmd_arch/x86/boot/bzImage咯？ 有这么奇葩的东西？ 你还真别说，世上就是有这么奇葩的事儿。当然这点暂时我还是不太确信。我先分享一下我的发现，大家看看对不对。

编译好bzImage后，有.cmd文件， arch/x86/boot/.bzImage.cmd。你打开一看。。。怎么样，惊呆了吧，这里面还真就定义了这个变量 cmd_$@。好吧，这内核藏得够深的！

那是在哪里包含的呢？ 在根目录Makefile中有这么一段，
```
# read all saved command lines

targets := $(wildcard $(sort $(targets)))
cmd_files := $(wildcard .*.cmd $(foreach f,$(targets),$(dir $(f)).$(notdir $(f)).cmd))

ifneq ($(cmd_files),)
  $(cmd_files): ;	# Do not try to update included dependency files
  include $(cmd_files)
endif
```

先放着了，回来再确认。

好了，饶了这么一大圈，再回过头来。这个命令就是比较了上次编译这个目标命令和这次使用的命令。如果不一样，则会返回非空，提示命令需要再执行。

好了，你看着累，我写着也累了。

###<font color=#8b0000>如果条件为真，即依赖或者规则变化的情况</font>
如果上面if语句的结果为真，那么会执行下面的命令。
```
	@set -e;                     \
	$(echo-cmd) $(cmd_$(1));     \
	printf '%s\n' 'cmd_$@ := $(make-cmd)' > $(dot-target).cmd)
```
 
 其中有三个重要的变量，

1. $(echo-cmd) 显示的内容，就是make的时候屏幕上打印  
2. \$(cmd_$(1) 这个是真要执行的命令
3. $(dot-target).cmd 这个就是之前看到的那个保存命令行的文件

```
echo-cmd = $(if $($(quiet)cmd_$(1)),\
	echo '  $(call escsq,$($(quiet)cmd_$(1)))$(echo-why)';)
```
这个显示得很巧妙，根据quiet变量的定义有两种情况：
如果quiet定义了，那么就是显示缩略形式。
如果没有定义，就显示详细的命令。

具体这个形式在make bzImage例子中，定义如下
```
quiet_cmd_image = BUILD   $@
cmd_image = $(obj)/tools/build $(obj)/setup.bin $(obj)/vmlinux.bin \
			       $(obj)/zoffset.h $@
```

那当quiet定义的时候，屏幕上就显示

> BUILD   arch/x86/boot/bzImage

好了，那来看看最后一个变量 
```
###
# Name of target with a '.' as filename prefix. foo/bar.o => foo/.bar.o
dot-target = $(dir $@).$(notdir $@)
```

这个dot-target就是目标文件的绝对路径（？）的文件名，加上一个"."。

So~ I guess you get it :)

# <font color=#8b0000>特殊目标和变量</font>
##<font color=#8b0000>Force目标</font> 
这是一个挺有意思的target。在我看来，make会自动按照依赖关系找到哪些目标是需要更新的。但是内核中，偏偏要加上这个目标来强制去执行某个规则。

```
PHONY += FORCE
FORCE:

# Declare the contents of the .PHONY variable as phony.  We keep that
# information in a variable so we can use it in if_changed and friends.
.PHONY: $(PHONY)
```

从定义上看，这就是一个没有依赖也没有执行语句的规则。当我们把他放在某个目标的依赖中，则这个目标将一定会被重新生成。（在本次make的范围内）。 请参考make 手册中的描述 [Force Target] [1]

看一下在内核中使用的例子。

```
$(obj)/bzImage: $(obj)/setup.bin $(obj)/vmlinux.bin $(obj)/tools/build FORCE
	$(call if_changed,image)
	@echo 'Kernel: $@ is ready' ' (#'`cat .version`')'
```
正常情况下，make bzImage，这条规则都会被执行。如果所有的依赖都没有更新，那就只会打印"Kernel: arch/x86/boot/bzImage is ready"。

当你把FORCE从这条规则中拿掉，那么在依赖没有更新时， 则不会打印这条输出。从这点可以看出，FORCE对编译过程产生的作用。

##<font color=#8b0000>arg-check和cmd_$@目标</font> 
这两个用于判断某个目标的编译命令是否有变化，定义在scripts/Kbuild.include 和 $(dot-target).cmd文件中。

详细可以参考本文if_changed_xxx 和 cmd-files小节。 

##<font color=#8b0000>cmd-files和targets目标</font> 

这两个变量定义在根目录Makefile中, 从现在理解来看 是为了arg-check和cmd_$@变量服务的。

额，发现这个变量在scripts/Makefile.build中也有定义，而且在我看的这个例子（make bzImage）中确实使用的这个地方的定义。感觉是在
> make -f scripts/Makefile.build obj=arch/x86/boot arch/x86/boot/bzImage
中定义了该变量。

这两处的定义基本一致，除了在include的时略有不同。（为啥要在两个地方定义呢？）

又ps，当你在此处打印该变量会发现，仅仅make bzImage其实会有很多target被打印到。那问题来了，需要被make -f scripts/Makefile.build这么多次么？

```
# read all saved command lines

targets := $(wildcard $(sort $(targets)))
cmd_files := $(wildcard .*.cmd $(foreach f,$(targets),$(dir $(f)).$(notdir $(f)).cmd))

ifneq ($(cmd_files),)
  $(cmd_files): ;	# Do not try to update included dependency files
  include $(cmd_files)
endif
```
归根到底就是包含了所有的cmd_files，那我们先来看看这cmd_files文件中到底是个啥。

```
~/git/linux$ cat .vmlinux.cmd
cmd_vmlinux := /bin/bash scripts/link-vmlinux.sh ld -m elf_x86_64 --emit-relocs --build-id
 
~/git/linux$ cat arch/x86/boot/.bzImage.cmd 
cmd_arch/x86/boot/bzImage := arch/x86/boot/tools/build arch/x86/boot/setup.bin arch/x86/boot/vmlinux.bin arch/x86/boot/zoffset.h arch/x86/boot/bzImage
```

可以看到，就是定义了对应的cmd_$@变量～

# <font color=#8b0000>一些重要的文件和关系</font>

##<font color=#8b0000>Makefile.build</font>

这个文件可以说是整个编译中，具体执行动作的核心文件。比如会在根Makefile中用到：

```
###
# Shorthand for $(Q)$(MAKE) -f scripts/Makefile.build obj=
# Usage:
# $(Q)$(MAKE) $(build)=dir
build := -f $(srctree)/scripts/Makefile.build obj

$(vmlinux-dirs): prepare scripts
	$(Q)$(MAKE) $(build)=$@
```

那接下来展开看一下这个文件包含的内容

###<font color=#8b0000>变量初始化</font>

截取一段，

```
# Init all relevant variables used in kbuild files so
# 1) they have correct type
# 2) they do not inherit any value from the environment
obj-y :=
obj-m :=
```

是不是觉得很亲切？ 对了，感觉就像是一个函数先上来初始化一下局部变量。

###<font color=#8b0000>引用Makefile.include</font>

这个文件定义了些基本的变量和规则，比如 if_changed ， build

###<font color=#8b0000>引用目标目录的Kbuild或者Makefile</font>

好了，这里要包含目标目录下的kbuild或者Makefile了。就好像引用了头文件后，要开始做本目标目录指定的事情了。Kbuild的优先级比Makefile的高，一般应该只会有其一。

值得一提的是，现在内核的编译系统已经非常完善。当需要新增文件加入内核或者模块中，只要修改Kbuild或者Makefile就可以了。

好了，来看一下大概会长什么样子。

```
"init/Makefile"

obj-y                          := main.o version.o mounts.o

mounts-y			:= do_mounts.o
mounts-$(CONFIG_BLK_DEV_RAM)	+= do_mounts_rd.o
mounts-$(CONFIG_BLK_DEV_INITRD)	+= do_mounts_initrd.o
mounts-$(CONFIG_BLK_DEV_MD)	+= do_mounts_md.o
```
```
"net/9p/Makefile"

obj-$(CONFIG_NET_9P) := 9pnet.o
obj-$(CONFIG_NET_9P_VIRTIO) += 9pnet_virtio.o
obj-$(CONFIG_NET_9P_RDMA) += 9pnet_rdma.o

9pnet-objs := \
	mod.o \
	client.o \
	error.o \
	util.o \
	protocol.o \
	trans_fd.o \
	trans_common.o \

9pnet_virtio-objs := \
	trans_virtio.o \

9pnet_rdma-objs := \
	trans_rdma.o \
```

看到里面的obj-y了么？ 或者在make menuconfig的时候，配成obj-m。是不是正好和初始化的变量对上？

###<font color=#8b0000>引用Makefile.lib</font>

这个文件也很重要，包括了对 obj-y 、obj-m变量的处理， 一些编译链接选项变量定义，还有一些规则。

在这里， 我着重突出一下这个文件中的第一个部分。

####<font color=#8b0000>分辨出sub-dir</font>

```
# Handle objects in subdirs
# ---------------------------------------------------------------------------
# o if we encounter foo/ in $(obj-y), replace it by foo/built-in.o
#   and add the directory to the list of dirs to descend into: $(subdir-y)
# o if we encounter foo/ in $(obj-m), remove it from $(obj-m)
#   and add the directory to the list of dirs to descend into: $(subdir-m)

# Determine modorder.
# Unfortunately, we don't have information about ordering between -y
# and -m subdirs.  Just put -y's first.
modorder	:= $(patsubst %/,%/modules.order, $(filter %/, $(obj-y)) $(obj-m:.o=.ko))

__subdir-y	:= $(patsubst %/,%,$(filter %/, $(obj-y)))
subdir-y	+= $(__subdir-y)
__subdir-m	:= $(patsubst %/,%,$(filter %/, $(obj-m)))
subdir-m	+= $(__subdir-m)
obj-y		:= $(patsubst %/, %/built-in.o, $(obj-y))
obj-m		:= $(filter-out %/, $(obj-m))

# Subdirectories we need to descend into

subdir-ym	:= $(sort $(subdir-y) $(subdir-m))
```

这个东西为什么重要呢？ 我们来看一下Makefile.build文件中靠后的一部分。

```
# Descending
# ------------------------------------

PHONY += $(subdir-ym)
$(subdir-ym):
	$(Q)$(MAKE) $(build)=$@
```

是不是觉得有点意思？ 这样就可以在Makefile/Kbuild文件中定义一个带有"/"的目标，内核就自动会进入下一层目录去编译～

####<font color=#8b0000>composite objects</font>

在内核makefile文件中，经常看到有带objs的变量。 比如上文中摘取的net/9p/Makefile中就有9pnet-objs。 以前只知道效果，就是后面跟的目标会合成一起生成这个9pnet.o。 那这个生成的部分工作就是在这里做的 -- 去找到这些带有-objs的目标。

```
# if $(foo-objs) exists, foo.o is a composite object
multi-used-y := $(sort $(foreach m,$(obj-y), $(if $(strip $($(m:.o=-objs)) $($(m:.o=-y))), $(m))))
multi-used-m := $(sort $(foreach m,$(obj-m), $(if $(strip $($(m:.o=-objs)) $($(m:.o=-y)) $($(m:.o=-m))), $(m))))
multi-used   := $(multi-used-y) $(multi-used-m)
single-used-m := $(sort $(filter-out $(multi-used-m),$(obj-m)))
```

当然，实际的工作还在Makefile.build中做，我们会在下面某节看到具体的规则。

###<font color=#8b0000>默认目标__build</font>

```
PHONY := __build
__build:


__build: $(if $(KBUILD_BUILTIN),$(builtin-target) $(lib-target) $(extra-y)) \
	 $(if $(KBUILD_MODULES),$(obj-m) $(modorder-target)) \
	 $(subdir-ym) $(always)
	@:
```

其实啥都没干，就是告诉make，需要干的工作是哪些。

###<font color=#8b0000>编译链接规则</font>

后面就是一些规则了，比如如何从c代码编译出一个obj文件。

```
# Built-in and composite module parts
$(obj)/%.o: $(src)/%.c $(recordmcount_source) $(objtool_obj) FORCE
	$(call cmd,force_checksrc)
	$(call if_changed_rule,cc_o_c)
```

但类似的规则在Makefile.lib中也有，不知道分在两个地方写的目的和意义。

来看两个比较重要的规则。

####<font color=#8b0000>builtin-target</font>

```
quiet_cmd_link_o_target = LD      $@
# If the list of objects to link is empty, just create an empty built-in.o
cmd_link_o_target = $(if $(strip $(obj-y)),\
		      $(LD) $(ld_flags) -r -o $@ $(filter $(obj-y), $^) \
		      $(cmd_secanalysis),\
		      rm -f $@; $(AR) rcs$(KBUILD_ARFLAGS) $@)

$(builtin-target): $(obj-y) FORCE
	$(call if_changed,link_o_target)
```

这个作用在什么地方呢？ 你看在编译过的内核目录下，有好多built-in.o文件吧， 他们就是用这个规则链接起来的。

####<font color=#8b0000>composite objects</font>

```
#
# Rule to link composite objects
#
#  Composite objects are specified in kbuild makefile as follows:
#    <composite-object>-objs := <list of .o files>
#  or
#    <composite-object>-y    := <list of .o files>
#  or
#    <composite-object>-m    := <list of .o files>
#  The -m syntax only works if <composite object> is a module
link_multi_deps =                     \
$(filter $(addprefix $(obj)/,         \
$($(subst $(obj)/,,$(@:.o=-objs)))    \
$($(subst $(obj)/,,$(@:.o=-y)))       \
$($(subst $(obj)/,,$(@:.o=-m)))), $^)

quiet_cmd_link_multi-y = LD      $@
cmd_link_multi-y = $(LD) $(ld_flags) -r -o $@ $(link_multi_deps) $(cmd_secanalysis)

quiet_cmd_link_multi-m = LD [M]  $@
cmd_link_multi-m = $(cmd_link_multi-y)

$(multi-used-y): FORCE
	$(call if_changed,link_multi-y)
$(call multi_depend, $(multi-used-y), .o, -objs -y)

$(multi-used-m): FORCE
	$(call if_changed,link_multi-m)
	@{ echo $(@:.o=.ko); echo $(link_multi_deps); \
	   $(cmd_undef_syms); } > $(MODVERDIR)/$(@F:.o=.mod)
$(call multi_depend, $(multi-used-m), .o, -objs -y -m)
```

上文中我们说net/9p/9pnet.o这个目标就是用这个规则生成的。说来也简单，就是把 9pnet-objs 变量中的所有.o文件链接一下。

从link_multi_deps的定义可以看出来，有三种写法 -objs, -y, -m。

[1]: https://www.gnu.org/software/make/manual/html_node/Force-Targets.html
[2]: http://blog.csdn.net/richardysteven/article/details/52930533