逐步地我们已经对kbuild有了一定的了解，知道了kbuild还会有自己定义的函数，以及这些函数会在一个类似c语言的头文件中定义。做了这么多准备，该是时候探索真正的代码编译了。先来看看一个.o文件是如何编译的。

比如我们拿这个文件来做例子：

```
make mm/mmu_gather.o
```

# single-build -- 第一个规则

在[kbuild系统浅析][4]中single-build部分，我们知道如果编译目标是单个的obj文件，采用的规则是single-no-ko.

```
$(single-no-ko): $(build-dir)
	@:
```

可以看到，这个规则实际没有具体命令，而是根据依赖build-dir来生成。

```
build-dir	:= .

$(build-dir): prepare
	$(Q)$(MAKE) $(build)=$@ need-builtin=1 need-modorder=1 $(single-goals)
```

看上去也不复杂，但是不知道具体是什么样子。我们在上面的代码上加一个#，来看看具体命令是什么。

```
$(build-dir): prepare
	#$(Q)$(MAKE) $(build)=$@ need-builtin=1 need-modorder=1 $(single-goals)
```

再运行一次

```
$ make mm/mmu_gather.o
  CHK     include/config/kernel.release
  CHK     include/generated/uapi/linux/version.h
  CHK     include/generated/utsrelease.h
  CHK     include/generated/timeconst.h
  CHK     include/generated/bounds.h
  CHK     include/generated/asm-offsets.h
  CALL    scripts/checksyscalls.sh
#@make -f ./scripts/Makefile.build obj=. need-builtin=1 need-modorder=1 ./mm/mmu_gather.o
```

怎么样，通过显示实际执行的命令，是不是你感觉熟悉了些？从显示出的命令来看，生成mm/mmu_gather.o这个目标文件是重新又调用了一次make，而这次使用的是script/Makefile.build这个规则文件，传入的参数是obj=.，目标还是mm/mmu_gather.o。

你看，是不是又清晰了一些？

## 展开那串命令行

刚才我们偷懒，直接打印出执行的命令。现在我们来看那串命令行究竟是如何展开的。

```
	$(Q)$(MAKE) $(build)=$@ need-builtin=1 need-modorder=1 $(single-goals)
```

第一招，我们先来拆分。这条命令中出现了好几个变量，Q, MAKE, build, single-goals。single-goals好理解就是我们的mm/mmu_gather.o。那实际上就剩这个build了。

通过我们比较丑的方法得到最后展开的命令来看，这个build变量包含的是

>  -f ./scripts/Makefile.build obj

关键它在哪里呢？还记得[上文][2]找过的那个“头文件”么？对了，还是在那scripts/Kbuild.include。

```
###
# Shorthand for $(Q)$(MAKE) -f scripts/Makefile.build obj=
# Usage:
# $(Q)$(MAKE) $(build)=dir
build := -f $(srctree)/scripts/Makefile.build obj
```

瞧，是不是和我们估计得长得差不多。

# single-target -- 第二个规则

终于把第一步的命令看明白了，现在是时候研究规则文件scripts/Makefile.build是如何编译出mm/mmu_gather.o的了。

```
@make -f ./scripts/Makefile.build obj=. need-builtin=1 need-modorder=1 ./mm/mmu_gather.o
```

也就是这么一个命令是怎么工作的。作为唯一的makefile，那就只能从Makefile.build入手了。

按照同样的方法在scripts/Makefile.build文件中找关键的部分：

```
# Single targets
# ---------------------------------------------------------------------------

single-subdirs := $(foreach d, $(subdir-ym), $(if $(filter $d/%, $(MAKECMDGOALS)), $d))
single-subdir-goals := $(filter $(addsuffix /%, $(single-subdirs)), $(MAKECMDGOALS))

$(single-subdir-goals): $(single-subdirs)
	@:
```

## subdir-ym的来历

要理解上面这段规则，就要从subdir-ym说起。

```
# Subdirectories we need to descend into
subdir-ym := $(sort $(subdir-y) $(subdir-m) \
			$(patsubst %/,%, $(filter %/, $(obj-y) $(obj-m))))
```

subdir-ym的定义是所有subdir-y, subdir-m和obj-y,obj-m中的目录部分。而且把后缀/去掉了。

那这些变量是哪里定义的呢？这就牵扯到了scripts/Makefile.build中的include $(kbuild-file)。但这个kbuild-file又是谁？这个又是在scripts/Kbuild.include中定义的。

```
###
# The path to Kbuild or Makefile. Kbuild has precedence over Makefile.
kbuild-file = $(or $(wildcard $(src)/Kbuild),$(src)/Makefile)
```

也就是make执行时对应的$(src)/Kbuild或者$(src)/Makefile文件。

值的注意的是，这里的路径$(src)是在Makefile.build开头定义的$(srcroot)/$(obj)。srcroot一般是内核根目录(参考[kbuild系统浅析][4])，obj我们传入的是"."。所以这里找到的就是根目录下的Kbuild文件。在这个文件的最后可以看到遗传obj-y/obj-m的定义。也就是可能依赖的所有子目录了。

绕了这么一大圈，我自己都已经晕了，让我们总结一下。

```
scripts/Makefile.build
-----------------------------------------
src := $(srcroot)/$(obj)                                          // 定义目录src

include $(srctree)/scripts/Kbuild.include
    kbuild-file = $(or $(wildcard $(src)/Kbuild),$(src)/Makefile) // 设置需要包含的Kbuild文件，在src目录下

include $(kbuild-file)                                            // 包含Kbuild文件，其中定义了obj-y/obj-m

subdir-ym := $(sort $(subdir-y) $(subdir-m) \                     // subdir-ym定义为obj-y/obj-m的集合，去掉/
		$(patsubst %/,%, $(filter %/, $(obj-y) $(obj-m))))

ifneq ($(obj),.)
subdir-ym	:= $(addprefix $(obj)/, $(subdir-ym))             // 下沉到子目录后，加上目录前缀，用于匹配目标
endif
```

现在再回过来看这两个变量的定义。

  * single-subdirs筛选出MAKECMDGOALS中目录和subdir-ym中匹配的目录
  * single-subdir-goals筛选出MAKECMDGOALS中带目录，且后面跟文件的目标

所以匹配完之后，我们看到这么一条规则

```
mm/mmu_gather.o: mm
	@:
```

ok，这就是我们期待的规则。慢着，这规则是啥？

# Descend -- 第三个规则

到这里又要开始找了，上面的规则断了线。还好，上面规则中还有一个依赖，mm。我们从这里再往下看看。

Makefile.build中有这么一个规则：

```
# Descending
# ---------------------------------------------------------------------------

PHONY += $(subdir-ym)
$(subdir-ym):
	$(Q)$(MAKE) $(build)=$@ \
	need-builtin=$(if $(filter $@/built-in.a, $(subdir-builtin)),1) \
	need-modorder=$(if $(filter $@/modules.order, $(subdir-modorder)),1) \
	$(filter $@/%, $(single-subdir-goals))
```

其中subdir-ym正好包含了我们的依赖mm。老办法，加上#看看效果。

我们得到了真正执行的命令：

```
make -f ./scripts/Makefile.build obj=mm \
need-builtin=1 \
need-modorder=1 \
mm/mmu_gather.o
```

难怪，subdir-ym加入了PHONY，所以虽然没有mm这个实际的目标，但是每次都会执行。

通过注释调这个规则，mmu_gather.o再也没有编译出来过。说明找对了。

## Descend递归

我们现在研究的目标是mm/mmu_gather.o，那如果是mm/kfence/core.o呢？这里，开发者充分使用了递归的思想。

如果目标是mm/kfence/core.o，执行上面这个make后，对应的kbuid-file就是mm/Makefile，而里面的obj-y就包含kfence。

此时又因为$(obj)不是"."了，所以subdir-ym里面的成员会加上$(obj)这个前缀，变成了mm/kfence。(这是关键)

这样进一步导致single-subdirs赋值为mm/kfence，single-subdir-goals赋值为mm/kfence/core.o。就会生成一个针对mm/kfence/core.o的single-subdir-goals规则，并且依赖subdir-yw，也就是mm/kfence。

这样就以obj=mm/kfence再调用一次make，进入到下层目录。到这里，就是一个递归了。

如果目标还有下层目录，就按照这种方式不断往下一层目录走，到最后用cc_o_c来编译。

# cc_o_c -- 最后一个规则

我想应该是最后了吧，这次make调用，已经下到了mm目录，且目标就是mmu_gather.o。

```
# Built-in and composite module parts
$(obj)/%.o: $(obj)/%.c $(recordmcount_source) FORCE
	$(call if_changed_rule,cc_o_c)
	$(call cmd,force_checksrc)
```

我想，终于是找到编译mmu_gather.o的最后那条规则了。

## 接近真相

上一节，我们把关注的焦点定在了这条语句。

```
    $(call if_changed_rule,cc_o_c)
```

是不是看着有点丈二和尚摸不少头脑？先别着急，我们还是一点点来分析。

call的用法在[上一篇][2]中也已经解释了，其实就是调用if_changed_rull这个变量。既然如此，那我们就来找找这个变量呗～

再去我们的头文件scripts/Kbuild.include中找找看。是不是踏破铁鞋无觅处，得来全不费功夫。

```
# Usage: $(call if_changed_rule,foo)
# Will check if $(cmd_foo) or any of the prerequisites changed,
# and if so will execute $(rule_foo).
if_changed_rule = $(if $(strip $(any-prereq) $(arg-check) ),      \
	@set -e;                                                      \
	$(rule_$(1)), @:)
```

嗯，是不是千万只草泥马又从心中奔腾而过了？刚才还觉得一切易如反掌，瞬间就变成了一头雾水。其实我的内心也是崩溃的，不过没办法，这文章都写了一大半了，总不能在这个时候停掉。自己挖的坑，硬着头皮也得把它填上了。

静下心来一看，这其实就是一个if语句。当条件为真，执行逗号之前的动作。当条件为假，则执行后面那个@:。然后你再对照一下注释，诶，还真是这么个理儿。关于后面这个@:，我也不是很确认是个神马东西。然后查了一下git记录，发现就是为了减少一些讨厌的输出消息的。具体说明可以看这个[kbuild: suppress annoying "... is up to date." message][1]。而后进一步发现shell中还有一个啥都不干的[冒号语句][3]。

感觉又有了一点点的小进步，是不是还是挺开心的。

这下我们把注意集中在rule_$(1)，仍然暂时先不看前置条件。刚才我们说了if_changed_rule是一个函数，传入的参数刚是cc_o_c。你看这时候正好用上，在这里展开后就是rule_cc_o_c。

这里有点像回调函数的感觉，if_changed_rule负责判断有没有前置条件是新的，是否需要重新生成目标。如果需要，if_changed_rule就会调用所要求的函数再去生成目标。

这一路走来真是不容易。给自己倒杯咖啡，听听音乐，歇一歇。马上就要看到最后的结果了。

## 云开日出

经过对if_changed_rule的分析，我们停在了rule_cc_o_c上。现在的任务就是找到它。

这次比较简单，定义就在scripts/Makefile.build中。

```
define rule_cc_o_c
	$(call echo-cmd,checksrc) $(cmd_checksrc)	  \
	$(call cmd_and_fixdep,cc_o_c)				  \
	$(cmd_modversions_c)						  \
	$(cmd_objtool)						          \
	$(call echo-cmd,record_mcount) $(cmd_record_mcount)
endef
```

又是一长串是不是？ 不过你有没有发现熟悉的cc_o_c的身影？而且也是作为一个自定义的函数的参数？那我们来猜一下，这个cmd_and_fixdep函数也会在某个地方调用参数所指定的一个函数来完成生成目标的动作。

那它在哪呢？ 对了，和if_changed_rule一样，也是在scripts/Kbuild.include文件中。

```
cmd_and_fixdep =                          \
	$(echo-cmd) $(cmd_$(1));              \
	...
```

为了不闹心，我就复制一下最关键的部分吧。你看，还是调用到了我们的参数。这次展开后就是cmd_cc_o_c了。它还是在scripts/Makefile.build文件中。

```
cmd_cc_o_c = $(CC) $(c_flags) -c -o $@ $<
```

好了，可以松一口气了。如果用过makefile的童鞋，看到这个是不是觉得很亲切？这就是我们平时手动编译的时候输入的命令。

> At last, you got it.

# 执行顺序总结

刚才在探索如何编译单个.o文件的道路上一路飞奔，虽然终于我们找到了那条最最最后的语句在哪里，但是估计我们自己的行囊丢了一地。中间打下的江山可能自己也记不太清楚了。

为了更好的理解kbuild，在下一次我们出发探索其他复杂的目标前，我们有必要在此总结这次探索的收获，就好像千里行军总得找个好地方，安营扎寨，整顿行囊，这样才能走得更远，更稳。

从最开始的Makefile开始，中间依赖了不少规则，是时候总结一下了。

```
    Makefile
    ---------------
    $(single-no-ko): $(build-dir)              // 因为是single target，但实际是通过依赖build-dir编译
        @:


    Makefile
    ---------------
    build-dir	:= .                           // 回过头看，从最上层就有递归的意思了。
    $(build-dir): prepare                      // 目标的依赖是一个目录，进入到目录用Makefile.build来构建。最开始都是根目录"."
        $(Q)$(MAKE) $(build)=$@ need-builtin=1 need-modorder=1 $(single-goals)


    scripts/Makefile.build
    ---------------
    $(single-subdir-goals): $(single-subdirs)  // 匹配上single-subdir-goals，但实际是通过依赖single-subdirs编译
        @:


    scripts/Makefile.build
    ---------------
    PHONY += $(subdir-ym)                      // single-subdirs 是 subdir-ym的一部分
    $(subdir-ym):                              // 和Makefile中的规则很像，进入到对应的目录构建。
        $(Q)$(MAKE) $(build)=$@ \
        need-builtin=$(if $(filter $@/built-in.a, $(subdir-builtin)),1) \
        need-modorder=$(if $(filter $@/modules.order, $(subdir-modorder)),1) \
        $(filter $@/%, $(single-subdir-goals))


    scripts/Makefile.build
    ---------------
    # Built-in and composite module parts      //通过递归，不断descend到目标文件锁在目录，最后真正从.c到.o。
    $(obj)/%.o: $(obj)/%.c $(recordmcount_source) FORCE
        $(call if_changed_rule,cc_o_c)
        $(call cmd,force_checksrc)


    scripts/Makefile.build                     // 具体的两条命令
    ---------------
    rule_cc_o_c
    	$(call cmd_and_fixdep,cc_o_c)


    scripts/Makefile.build
    ---------------
    cmd_cc_o_c
		$(CC) $(c_flags) -c -o $@ $<
```

这么看或许可以更清楚一些。对单个.o文件的编译，一共是四个步骤。在我们平时写的makefile中，可能到第二步就可以直接写上cmd_cc_o_c的命令了。而在内核中，又多出了两步为了检查有没有依赖关系的更新及其他的一些工作。

在探索的过程中或许会有总也没有尽头的感觉，而现在整理完一看，一共也就四个层次，是不是觉得好像也没有那么难了？是不是觉得不过如此了？

[1]: https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=2aedcd098a9448b11eab895ee79acf519686555a
[2]: /brief_tutorial_on_kbuild/04_one_example_of_kbuild_function_cscope.md
[3]: http://linuxcommand.org/man_pages/cd1.html
[4]: /brief_tutorial_on_kbuild/13_root_makefile.md
