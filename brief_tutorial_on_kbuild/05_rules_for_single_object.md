逐步地我们已经对kbuild有了一定的了解，知道了kbuild还会有自己定义的函数，以及这些函数会在一个类似c语言的头文件中定义。做了这么多准备，该是时候探索真正的代码编译了。先来看看一个.o文件是如何编译的。

比如我们拿这个文件来做例子：

```
make mm/memblock.o
```

# 找到我们的目标

有了刚才的经验，这次我们顺着之前的思路。先到根目录的Makefile中找找线索。有时候找代码除了经验，还得有点运气。不知道你这次有没有找到呢？

单个文件的目标还是在根目录Makefile文件中。

```
%.o: %.c prepare scripts FORCE
	$(Q)$(MAKE) $(build)=$(build-dir) $(target-dir)$(notdir $@)
```

你看，这格式其实和我们平时自己写的规则是差不多的。.o的文件依赖于同名的.c文件，然后执行了一个命令来生成。不过呢，确实还多了些东西，包括一些特殊的依赖条件，以及这个命令长得也有点丑。

俗话说的好，恶人自有恶人磨，长得丑的代码也有丑办法来解读。

把上面这段代码添加一个井号

```
%.o: %.c prepare scripts FORCE
	#$(Q)$(MAKE) $(build)=$(build-dir) $(target-dir)$(notdir $@)
```

再运行一次

```
$ make mm/memblock.o
  CHK     include/config/kernel.release
  CHK     include/generated/uapi/linux/version.h
  CHK     include/generated/utsrelease.h
  CHK     include/generated/timeconst.h
  CHK     include/generated/bounds.h
  CHK     include/generated/asm-offsets.h
  CALL    scripts/checksyscalls.sh
# @make -f ./scripts/Makefile.build obj=mm mm/memblock.o
```

怎么样，通过显示实际执行的命令，是不是你感觉熟悉了些？从显示出的命令来看，生成mm/memblock.o这个目标文件是重新又调用了一次make，而这次使用的是script/Makefile.build这个规则文件，传入的参数是obj=mm，目标还是mm/memblock.o。

你看，是不是又清晰了一些？

# 展开那串命令行

现在我们来看那串命令行究竟是如何展开的。

```
    $(Q)$(MAKE) $(build)=$(build-dir) $(target-dir)$(notdir $@)
```

第一招，我们先来拆分。这条命令中出现了好几个变量，Q, MAKE, build, build-dir, target-dir。前面两个就是定义成@和make的，基本问题不大也比较好理解。关键是后面三个。

## build-dir 和 target-dir

可巧，最后两个变量的定义就在这组规则的上方。我们拿来先看一下。

```
# Single targets
# ------------------------------------------------------------
# Single targets are compatible with:
# - build with mixed source and output
# - build with separate output dir 'make O=...'
# - external modules
#
#  target-dir => where to store outputfile
#  build-dir  => directory in kernel source tree to use

ifeq ($(KBUILD_EXTMOD),)
        build-dir  = $(patsubst %/,%,$(dir $@))
        target-dir = $(dir $@)
else
        zap-slash=$(filter-out .,$(patsubst %/,%,$(dir $@)))
        build-dir  = $(KBUILD_EXTMOD)$(if $(zap-slash),/$(zap-slash))
        target-dir = $(if $(KBUILD_EXTMOD),$(dir $<),$(dir $@))
endif
```

看注释，这两个变量一个是做编译的路径，一个是目标存放的路径。

定义本身分为两种情况
* 当KBUILD_EXTMOD为空，表示不是编译内核模块
* 当KBUILD_EXTMOD不为空，表示是在编译内核模块

这次我们并没有编译内核模块，所以采用的是第一种情况的定义。从显示出的命令来看build-dir和target-dir分别定义为mm和mm/。说明指向的路径是一样的，只是相差了最后的一个/。

## build

五个变量基本理解了四个，那现在来看看最后一个build。

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

# 再进一步

终于把第一步的命令看明白了，现在是时候研究规则文件scripts/Makefile.build是如何编译出mm/memblock.o的了。

按照同样的方法在scripts/Makefile.build文件中找.o的目标。找啊找啊找，终于看到一条像的了。

```
# Built-in and composite module parts
$(obj)/%.o: $(src)/%.c $(recordmcount_source) $(objtool_obj) FORCE
	$(call cmd,force_checksrc)
	$(call if_changed_rule,cc_o_c)
```

嗯，究竟是不是这条规则？ 还记得我们的丑办法么？留给大家自己试验一次～

依赖条件我们暂时也不看了，不是关注我们的目标--.o文件是怎么编译出来的。看这个规则中的两条命令，第一条是做代码检查的，那暂时也不关注吧。第二条看着名字就有点像，cc_o_c，把c代码通过cc制作成o文件。这个正是我们要关注的，就分析它了。

# 接近真相

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

# 云开日出

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

# 整顿行囊

刚才在探索如何编译单个.o文件的道路上一路飞奔，虽然终于我们找到了那条最最最后的语句在哪里，但是估计我们自己的行囊丢了一地。中间打下的江山可能自己也记不太清楚了。

为了更好的理解kbuild，在下一次我们出发探索其他复杂的目标前，我们有必要在此总结这次探索的收获，就好像千里行军总得找个好地方，安营扎寨，整顿行囊，这样才能走得更远，更稳。

## 执行顺序
从文件执行顺序上整理如下：

```
    Makefile
    ---------------
    %.o: %.c
    	make -f scripts/Makefile.build obj=mm mm/memblock.o

    scripts/Makefile.build
    ---------------
    $(obj)/%.o: $(src)/%.c
    	$(call if_changed_rule,cc_o_c)

    scripts/Makefile.build
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

## scripts/Makefile.build

这个文件几乎包含了所有重要的规则，rule_cc_o_c, cmd_cc_o_c都在这个文件中定义。以后我们会看到，凡事要进行编译工作，都会使用这个规则文件。

## scripts/Kbuild.include

这个文件包含了一些有意思的函数，if_changed_rule, cmd_and_fixdep。而且这个文件被根目录Makefile和scripts/Makefile.build都包含使用。

好了，这下真的要歇一歇了。吃个大餐慰劳一下自己吧～

[1]: https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=2aedcd098a9448b11eab895ee79acf519686555a
[2]: http://blog.csdn.net/richardysteven/article/details/56833038
[3]: http://linuxcommand.org/man_pages/cd1.html
