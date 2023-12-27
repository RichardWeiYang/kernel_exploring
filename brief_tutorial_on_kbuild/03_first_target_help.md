help目标可以说是在kbuild中最直接的小目标了，虽然它和我们的代码基本没有什么关系，而是用来生成kbuild的简短使用说明，但是用它来作为走近kbuild系统的敲门砖，或许是比较合适的。

# 如何用？

用法还是很简单的

```
make help
```

就可以显示出当前kbuild所支持的小目标们。比如：

* vmlinux
* modules
* clean
* dir/file.o

具体的大家可以动手操作，体会一下这个过程。

# 在哪里？

万事开头难，既然是你遇到的头一个目标，可能你会丈二和尚摸不着头脑。不知道会是在哪里。

> 肿么办？

如果是你，你会想怎么做呢？请在这里停留一分钟，自己思考一下再往下看我提供的做法。

**这里是分割线**
---

你看，我们平时自己使用make的时候是怎么用的呢？ 要写makefile是吧，在makefile里面加上目标和规则是吧。那好了，kbuild也是基于make这个基本结构运作的。那就是找到别人写的那个makefile呗。

先别着急找，我们先来看一下make的手册是怎么讲的。

> Once a suitable makefile exists, each time you
> change some source files, this simple shell 
> command:
> 
>      make
> 
> suffices to perform all necessary recompilations.  
> The make program uses the makefile description 
> and the  last-modification  times  of  
> the files to decide which of the files need 
> to be updated.  For each of those files, it 
> issues the commands recorded in the makefile.
> 
> make executes commands in the makefile to 
> update one or more target names, where name 
> is  typically  a  program. If no -f option 
> is present, make will look for the makefiles 
> GNUmakefile, makefile, and Makefile, in that 
> order.
 
总结一下：

* 运行make后，会去寻找makefile，根据其中的规则做更新
* 可以使用选项-f指定要寻找那个makefile，如果没有指定则按照上述顺序去寻找

还稍微需要解释一下下，这里的makefile这个词有两种不同的意义，头一次看的估计会晕，说实话我也有点晕。
* 代词，代表的是make使用的规则文件，并不是具体的哪个文件。
* 名字，是指make运行时，如果没有传入-f选项，那么会按照GNUmakefile, makefile, Makefile这个顺序去搜索规则文件

从上面的手册中，我们可以看到，运行make其实是又其自身的要求的。也就是需要有个规则文件。那我们再来做个实验。

随便新建一个目录，cd进去，运行make，看一下结果。

```
$ mkdir test
$ cd test/
$ make
make: *** No targets specified and no makefile found.  Stop.
```

你看是不是啥都干不了？ 

整理了一下make的基本知识，再回过来看我们执行的命令。

```
make help
```

这次我们的make命令并没有带选项-f，所以按照手册所说，应该是在本地按照顺序寻找了规则文件再执行的。 那我们来看一下，内核源码根目录下都有谁呗。

```
ls
OPYING        REPORTING-BUGS include        scripts
CREDITS        arch           init           security
Documentation  block          ipc            sound
Kbuild         certs          kernel         tools
Kconfig        crypto         lib            usr
MAINTAINERS    drivers        mm             virt
Makefile       firmware       net
README         fs             samples
```

我相信你已经看到了点什么。 正所谓，

> 众里寻她千百度，蓦然回首，那人却在，灯火阑珊处。

# 什么样？

已经找到了规则文件Makefile， 那我们就打开看一下，找找我们的help小目标呗。

相信你已经看到了～ 它就长这个样子：

```
help:
	@echo  'Cleaning targets:'
	@echo  '  clean		  - Remove most generated files but keep the config and'
	@echo  '                    enough build support to build external modules'
	@echo  '  mrproper	  - Remove all generated files + config + various backup files'
	@echo  '  distclean	  - mrproper + remove editor backup and patch files'
	@echo  ''
	@echo  'Configuration targets:'
	@$(MAKE) -f $(srctree)/scripts/kconfig/Makefile help
	@echo  ''
	@echo  'Other generic targets:'
	@echo  '  all		  - Build all targets marked with [*]'
	@echo  '* vmlinux	  - Build the bare kernel'
	@echo  '* modules	  - Build all modules'
	@echo  '  modules_install - Install all modules to INSTALL_MOD_PATH (default: /)'
	...
```

怎么样，确实够直接吧，在根目录的Makefile中就找到了目标。看来我们今天的运气还不错～

# 恭喜你

恭喜，你已经知道了一个kbuild的小目标是如何运作起来的了。你看是不是和我们平时见到的最简单的makefile结构差不多呢？

一切事物皆有源头，哪怕是再复杂的结构都可以将其拆分成简单的组成部分，而去逐个了解和研究。我们的kbuild更是如此。相信你可以通过不断探索，掌握这看似庞大的kbuild系统～

祝好，加油～
