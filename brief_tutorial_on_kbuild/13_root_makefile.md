编译内核的这一套叫做kbuild，是在makefile的基础上，为了方便和统一内核编译搭建的一套自成体系的系统。我们现在从整体结构上来学习以下kbuild系统。

# 根Makefile的结构

首先我们从根Makefile开始，因为最基本的一切都是从根Makefile开始的。


```
this-makefile := $(lastword $(MAKEFILE_LIST))
abs_srctree := $(realpath $(dir $(this-makefile)))
abs_output := $(CURDIR)

sub_make_done != 1     // 第一次一定会被执行，用来设置参数
    设置了一些参数，如：
    KBUILD_EXTMOD := $(M)  真正干活的时候，会以这个作区分，执行的操作不一样
    abs_output如果用户指定输出目录，会变化。影响到下面的need_sub_make。
    export sub_make_done := 1
end

// 根据目标输出目录是否是当前目录，判断是否要嵌套执行
ifneq ($(abs_output),$(CURDIR))
need-sub-make := 1
endif

need-sub-make == 1
        // 如果目标输出目录核当前目录不一样，就重新执行一次make
        $(Q)$(MAKE) $(no-print-directory) -C $(abs_output) \
        -f $(abs_srctree)/Makefile $(MAKECMDGOALS)
else

        // 接下来判断是不是要逐个make

        设置下面几个参数，决定这次构建的方式
        config-build     :=
        mixed-build      :=
        need-config      := 1
        may-sync-config  := 1
        single-build     :=

        // 根据MAKECMDGOALS来判断是否需要设置上面的值，来决定接下来做什么共作
        mixed-build == 1
            对$(MAKECMDGOALS)中的目标，依次执行
            make -f $(srctree)Makefile $$i
        else
            // 到这里才开始真正干活

            包含kbuild核心文件，其中定义了build := -f $(srctree)/scripts/Makefile.build obj
            include scripts/Kbuild.include

            // out of tree规则
            ifdef building_out_of_srctree
            endif

            // 目标是不是config文件
            config-build
                构建配置，如make menuconfig
            else
                // 读取配置，看上去和.config一样
                include include/config/auto.conf

                // 如果需要.config但是没有，报错

                // 根据配置include需要的Makefile

                // 先是内核的规则，后是模块的规则
                if !KBUILD_EXTMOD
                    build-dir := .
                else
                endif

                // 如果有single target, 如.o, mm/等
                if single-build
                endif

                $(build-dir): prepare
                	$(Q)$(MAKE) $(build)=$@ need-builtin=1 need-modorder=1 $(single-goals)
            end
        end
end  # need-sub-make
```

我把根Makefile的骨架子，通过注释的方式列了出来。感觉终于对根Makefile有了点了解。

首先根Makefile会判断以下是否需要重新执行一下make -f Makefile。接下来会根据编译目标，MAKECMDGOALS来区分，包括

* config-build
* single-build
* mixed-build

其中config-build就是生成.config配置的。mixed-build是有混合目标时触发的，如同时有config/clean/真实目标，kbuild会把他们分开，依次执行make。single-build比较特殊，单独划出了一类目标处理。这么一看感觉两千多行的Makefile，也不是那么晦涩了。

## 小tip

kbuild涉及的文件较多，还有条件判断，这样导致目标的依赖不一定很清楚。而make -p可以把整个编译的目标、依赖和命令都输出出来。可以帮助我们理解构建时的具体内容。

比如

>> make -p modules

可以打印出目标是modules时所有的变量和规则定义。

不过这个输出还是有点大的。。。

## 获取当前目录

内核在编译时会包含多个makefile，所以根Makefile通过MAKEFILE_LIST来获取当前执行时的目录。

```
this-makefile := $(lastword $(MAKEFILE_LIST))
abs_srctree := $(realpath $(dir $(this-makefile)))
abs_output := $(CURDIR)
```

MAKEFILE_LIST保存了读取的所有makefile，最后一个是当前的。通过这种方式来得到当前真实目录。

## sub_make判断

这是根Makefile的第一部分，其中我们又可以分成两部分：

  * sub_make_done
  * need-sub-make

主要是解决嵌套执行的。

变量sub_make_done控制了一些变量只要设置一次。设置完后，sub_make_done会设置为1并export，表示以后不会再被运行。

这部分设置的变量比如有：

  * quiet/Q/KBUILD_VERBOSE: 用来控制编译时输出是否简化
  * KBUILD_EXTMOD: 是否用了make M=dir，将dir设置到变量KBUILD_EXTMOD，并export
  * output/abs_output: 编译结果存放的目录/绝对目录，默认是执行make的当前目录

设置完后，根据abs_output和CURDIR的值是否相等设置need-sub-make。如果不相等说明需要嵌套执行make，会重新调用一遍make。如果相等，那接下来才是正经的make工作。

在研究真正的工作前，我们看看sub_make是怎么做的。

```
        $(Q)$(MAKE) $(no-print-directory) -C $(abs_output) \
        -f $(abs_srctree)/Makefile $(MAKECMDGOALS)
```

其中：

  * -C 表示先进入到对应的目录，再执行make，而且CURDIR被设置为该目录，而不是执行make时的目录
  * -f 使用哪个makefile，这个会出现在MAKEFILE_LIST中

这时我们再来看Makefile开头的变量定义：

```
this-makefile := $(lastword $(MAKEFILE_LIST))
abs_srctree := $(realpath $(dir $(this-makefile)))
abs_output := $(CURDIR)
```

结合上面的命令行我们可以得到：

  * 源代码目录由-f传入的Makefile指定
  * 输出结果目录由-C传入的目录指定

当然对输出结果的目录还有一个小插曲，就是可以通过-O选项来指定输出目录。好在这一切都发生在sub_make前，也就是当我们执行这个嵌套make的时候已经决定好了。

### 几个用到的变量

在sub make阶段，以及刚开始处理的时候，由几个很相似的变量让我困惑。

  * abs_srctree: 这个变量只有在最开头的时候设置过，后面不再变化。它被设置为Makefile所在的目录，一般就是内核源码的根目录
  * abs_output: 这个变量含义时编译输出结果保存到哪个目录。实际上经过sub make后，就一直是CURDIR，也不会再变了
  * srcroot: 如果有M选项，就是这个模块的目录；如果没有就是内核根目录
  * srctree: 如果有M选项，就是内核根目录；如果没有就是srcroot(也是内核根目录？)

## single-build

如果命令行中指定了单个要编译的目标，如 xxx.o，那么规则就会走到这里，由这里来处理。

这部分由下面代码来检测：

```
single-targets := %.a %.i %.ko %.lds %.ll %.lst %.mod %.o %.rsi %.s %/
...
ifneq ($(filter $(single-targets), $(MAKECMDGOALS)),)
    single-build := 1
    ...
endif
```

所以当我们编译单个目标时，就到这里来看规则。值的注意的是make mm/也算是单个目标。

但是这里出现了一个神奇的地方。

```
$(single-no-ko): $(build-dir)
	@:
```

这个是一个空规则。。。。所以到底是怎么编译这个单个目标的呢？ 玄机隐藏再build-dir中。

```
$(build-dir): prepare
	$(Q)$(MAKE) $(build)=$@ need-builtin=1 need-modorder=1 $(single-goals)
```

在Makefile中有一个针对build-dir的规则，实际上的单个目标是通过这条命令编译的。比如我们要编译mm/mmu_gather.o，展开后是这样。

```
make -f ./scripts/Makefile.build obj=. need-builtin=1 need-modorder=1 ./mm/mmu_gather.o
```

到这里还没有结束，接下去的深入探索我们放到[单个.o文件的编译][1]中。

# kbuild

* Makefile.build文件
* 常用函数

## 从Makefile.build开始

在上面我摘出来的根Makefile文件末尾，我保留了一段具体的目标规则的代码。这部分，就是内核编译过程中干活的核心。

在内核编译文件内，我们可以看到很多类似下面的代码：

>> $(Q)$(MAKE) $(build)=dir

其中build是在Kbuild.include定义的变量。

>> build := -f $(srctree)/scripts/Makefile.build obj

所以在调用时，真正执行的是

>> make -f scripts/Makefile.build obj=dir

所以内核编译时，在继承了根Makefile导出的变量设置后，具体的工作交给了每个单独的Makefile.build来完成。

了解了这个调用方式后，就来看看这个Makefile.build的庐山真面目。

```
src := $(obj)

默认目标
$(obj)/:

-include include/config/auto.conf

include scripts/Kbuild.include
    设置kbuild-file变量，接下来将被引用。 srctree在根Makefile中定义
    kbuild-dir = $(if $(filter /%,$(src)),$(src),$(srctree)/$(src))
    kbuild-file = $(or $(wildcard $(kbuild-dir)/Kbuild),$(kbuild-dir)/Makefile)
include $(kbuild-file)
    定义了obj-y,obj-m等变量
include scripts/Makefile.lib
    根据kbuild-file设置了subdir-ym

设置targets-for-modules，targets-for-builtin变量

默认目标依赖了subdir-ym
$(obj)/: $(if $(KBUILD_BUILTIN), $(targets-for-builtin)) \
	 $(if $(KBUILD_MODULES), $(targets-for-modules)) \
	 $(subdir-ym) $(always-y)
	@:

对每个subdir-ym，再执行make完成递归
PHONY += $(subdir-ym)
$(subdir-ym):
	$(Q)$(MAKE) $(build)=$@ \
	need-builtin=$(if $(filter $@/built-in.a, $(subdir-builtin)),1) \
	need-modorder=$(if $(filter $@/modules.order, $(subdir-modorder)),1) \
	$(filter $@/%, $(single-subdir-goals))
```

从上面的片段看，Makefile.build不是一个人在战斗。还有他的同伴Kbuild.include,Makefile.lib在一起协作。

每次调用Makefile.build时，就会去obj指定的目录下找到Kbuild或者Makefile。而这两个文件中就是按照kbuild定义的目标obj-y/obj-m。如果是目录，就会递归得进入下层目录继续编译。并且因为subdir-ym是默认目标的依赖，所以保证了下层目标会先生成。

## 层次结构

整个内核代码从kbuild的角度来看，可能是这样的。

```
     +--  Kbuild
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
```

每个层级的Kbuild/Makefile定义了obj-y/obj-m，如果需要则先进入下层目录，形成递归。

## 常用函数

scripts/Kbuild.include

### if_changed

内核Makefile中常见的构建方法就是

>> $(call if_changed,xxx)

这个函数的作用是判断目标的依赖是否有变化，如果有变化，则调用xxx函数进行构建。

来看一下定义

```
# Execute command if command has changed or prerequisite(s) are updated.
if_changed = $(if $(if-changed-cond),$(cmd_and_savecmd),@:)

cmd_and_savecmd =                                                            \
	$(cmd);                                                              \
	printf '%s\n' 'savedcmd_$@ := $(make-cmd)' > $(dot-target).cmd

```

就是当if-changed-cond不是空，就执行cmd_and_savecmd。也就是调用了cmd后，再把构建当前目标的命令保存到一个临时文件里。比如查看.vmlinux.o.cmd，就可以知道在链接vmlinux.o时的具体命令。

接下来就是看这个if-changed-cond是如何判断命令行和依赖是否有更新。

```
if-changed-cond = $(newer-prereqs)$(cmd-check)$(check-FORCE)
```

这里判断了依赖和命令行有没有变化。最后这个FORCE的作用还不是很清楚。

[1]: /brief_tutorial_on_kbuild/05_rules_for_single_object.md
