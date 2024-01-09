编译内核的这一套叫做kbuild，是在makefile的基础上，为了方便和统一内核编译搭建的一套自成体系的系统。我们现在从整体结构上来学习以下kbuild系统。

# 根Makefile的结构

首先我们从根Makefile开始，因为最基本的一切都是从根Makefile开始的。


```
!sub_make_done
    设置了一些参数，如：
    KBUILD_EXTMOD := $(M)  真正干活的时候，会以这个作区分，执行的操作不一样
    sub_make_done := 1
end

need_sub_make
    某些情况下会重新执行一下make
    make -C dir -f Makefile $(MAKECMDGOALS)
end

设置下面几个参数，决定这次构建的方式
config-build :=
single-build :=
mixed-build  :=

mixed-build
    对$(MAKECMDGOALS)中的目标，依次执行make -f Makefile $$i
else
    到这里才开始真正干活

    kbuild核心文件
    include scripts/Kbuild.include

    config-build
        构建配置，如make menuconfig
    else
        读取配置，看上去和.config一样
        include include/config/auto.conf

        !KBUILD_EXTMOD
            build-dir := .
        else
            build-dir := $(KBUILD_EXTMOD)
        end

        PHONY += $(build-dir)
        $(build-dir): prepare
        	$(Q)$(MAKE) $(build)=$@ need-builtin=1 need-modorder=1 $(single-goals)

    end
end
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

include include/config/auto.conf

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

