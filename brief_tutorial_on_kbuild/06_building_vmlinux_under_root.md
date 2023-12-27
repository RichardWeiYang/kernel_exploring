有编译过内核的话，一般都会看到在根目录下有个文件vmlinux，这个就是通常所说的内核了。但是用了这么久，倒是从来没看过是怎么编译出来的。那今天我们就通过它来试试我们这知识到底有没有掌握。

# 那些七大姑八大姨们

一切的一切都是make读取makefile编译链接的，就好像孙悟空逃不出如来佛祖的手掌，vmlinux的出世也是在makefile的安排之下。那就现在看看makefile

```
# SHELL used by kbuild
CONFIG_SHELL := $(shell if [ -x "$$BASH" ]; then echo $$BASH; \
	  else if [ -x /bin/bash ]; then echo /bin/bash; \
	  else echo sh; fi ; fi)

# Final link of vmlinux
      cmd_link-vmlinux = $(CONFIG_SHELL) $< $(LD) $(LDFLAGS) $(LDFLAGS_vmlinux)
quiet_cmd_link-vmlinux = LINK    $@

vmlinux: scripts/link-vmlinux.sh vmlinux_prereq $(vmlinux-deps) FORCE
	+$(call if_changed,link-vmlinux)
```

第一次看完这一段差点一口老血吐出来，我真是没有想到他们尽然可以这么玩。。。还好我们已经经历过了之前的九九八十一难，见到这些也还能算是气定神闲了～

if_changed也是在头文件scripts/Kbuild.include中定义，详细解释可以看[if_changed][1]。这个的用法也是类似之前看到的if_changed_rule是一个回调函数，当条件满足就运行cmd_link-vmlinux。 而这个cmd_link-vmlinux就是把第一个依赖作为脚本传给了系统使用的shell，由系统shell执行。好吧，我也是醉了，不过道行也就这么又深了一点点。你说是不是。

回到正题，这次我们先看看vmlinux相关依赖的都是谁。

vmlinux一共依赖两个 一个是vmlinux_prereq，另一个是vmlinux-deps。那分头分析。

## vmlinux_prereq

```
# Include targets which we want to execute sequentially if the rest of the
# kernel build went well. If CONFIG_TRIM_UNUSED_KSYMS is set, this might be
# evaluated more than once.
PHONY += vmlinux_prereq
vmlinux_prereq: $(vmlinux-deps) FORCE
ifdef CONFIG_HEADERS_CHECK
	$(Q)$(MAKE) -f $(srctree)/Makefile headers_check
endif
ifdef CONFIG_BUILD_DOCSRC
	$(Q)$(MAKE) $(build)=Documentation
endif
ifdef CONFIG_GDB_SCRIPTS
	$(Q)ln -fsn `cd $(srctree) && /bin/pwd`/scripts/gdb/vmlinux-gdb.py
endif
ifdef CONFIG_TRIM_UNUSED_KSYMS
	$(Q)$(CONFIG_SHELL) $(srctree)/scripts/adjust_autoksyms.sh \
	  "$(MAKE) KBUILD_MODULES=1 -f $(srctree)/Makefile vmlinux_prereq"
endif
```

这个看觉和最后的vmlinux关系不大， 咱暂时就不看了。（让哥偷个懒）

## vmlinux-deps

```
vmlinux-deps := $(KBUILD_LDS) $(KBUILD_VMLINUX_INIT)   $(KBUILD_VMLINUX_MAIN)
```
vmlinux-deps是这几个变量的集合，这下再次展开，我们一个个来看～

### KBUILD_LDS -- 链接文件

```
export KBUILD_LDS          := arch/$(SRCARCH)/kernel/vmlinux.lds
```

这个好说，就是链接文件～好多秘密都在这里，暂且不表，之后再说。

### KBUILD_VMLINUX_INIT

```
export KBUILD_VMLINUX_INIT := $(head-y) $(init-y)
```

KBUILD_VMLINUX_INIT又是两个变量的集合， head-y 和 init-y。

#### head-y

这个head-y定义根据架构不同而不同，比如在x86架构下，该定义为：

````
head-y := arch/x86/kernel/head_$(BITS).o
head-y += arch/x86/kernel/head$(BITS).o
head-y += arch/x86/kernel/head.o
````

恩，就一个架构，都还有三个文件，好多～忘了说了，这几个定义在arch/x86/Makefile中。那是怎么被根目录的Makefile引用到的呢？你猜？

在搜索的时候又发现这么一句在文档中：

> $(head-y) lists objects to be linked first in vmlinux.

这个提醒了我一下，在链接的时候，文件出现的顺序是有影响的。

#### init-y

```
init-y      := init/

init-y      := $(patsubst %/, %/built-in.o, $(init-y))
```

init-y有两种情况

* 是init/这个目录
* 是init/built-in.o这个文件。

在不同情况下展现出不同形式。而在KBUILD_VMLINUX_INIT这个变量中，保存的是init/built-in.o。

### KBUILD_VMLINUX_MAIN


```
export KBUILD_VMLINUX_MAIN := $(core-y) $(libs-y) $(drivers-y) $(net-y) $(virt-y)
```

这个包含的内容比较多了啊。

```
#core-y
core-y      := usr/
core-y      += kernel/ certs/ mm/ fs/ ipc/ security/ crypto/ block/
core-y      := $(patsubst %/, %/built-in.o, $(core-y))


#libs-y
libs-y      := lib/
libs-y1     := $(patsubst %/, %/lib.a, $(libs-y))
libs-y2     := $(patsubst %/, %/built-in.o, $(libs-y))
libs-y      := $(libs-y1) $(libs-y2)


#drivers-y
drivers-y   := drivers/ sound/ firmware/
drivers-y   := $(patsubst %/, %/built-in.o, $(drivers-y))

#net-y
net-y       := net/
net-y       := $(patsubst %/, %/built-in.o, $(net-y))


#virt-y
virt-y      := virt/
virt-y      := $(patsubst %/, %/built-in.o, $(virt-y))
```

这下，内核根目录下所有的子目录就都包含了～

除了lib目录下有两个文件，built-in.o 和 lib.a，其余下都只有一个文件built-in.o。到这里你有没有感觉眼前一亮？有没有觉得这个built-in.o很眼熟？

#### VMLINUX_MAIN中目标是如何的编译

可以看到VMLINUX_MAIN中包含的都是内核子系统的根目标了。我们可以猜到最终根目录下的vmlinux就是由这些根目标链接而成。那这些子系统的根目标是如何生成的呢？

过程有点绕，我们还是直接来看代码。

```
"Makefile"

vmlinux-deps := $(KBUILD_LDS) $(KBUILD_VMLINUX_INIT) $(KBUILD_VMLINUX_MAIN)

# The actual objects are generated when descending,
# make sure no implicit rule kicks in
$(sort $(vmlinux-deps)): $(vmlinux-dirs) ;
```

凡是在vmlinux-deps中的变量，都依赖于vmlinux-dirs变量的值。那就去看看vmlinux-dirs中的目标是什么规则咯。

```
vmlinux-dirs    := $(patsubst %/,%,$(filter %/, $(init-y) $(init-m) \
             $(core-y) $(core-m) $(drivers-y) $(drivers-m) \
             $(net-y) $(net-m) $(libs-y) $(libs-m) $(virt-y)))

# Handle descending into subdirectories listed in $(vmlinux-dirs)
# Preset locale variables to speed up the build process. Limit locale
# tweaks to this spot to avoid wrong language settings when running
# make menuconfig etc.
# Error messages still appears in the original language

PHONY += $(vmlinux-dirs)
$(vmlinux-dirs): prepare scripts
    $(Q)$(MAKE) $(build)=$@
```

vmlinux-dirs定义为目录。而他的规则是$(Q)$(MAKE) $(build)=$@。按照在scripts/Kbuild.include中的定义，展开后就是

> make -f scripts/Makefile.build obj=$@

他的意思就是，各回各家，各找各妈，各自去各自的目录去执行编译吧～

# 谁是你们的粘合剂

看到了这么多内核子系统的根目标，现在该是时候把他们链接起来生成vmlinux了～

还记得cmd_link-vmlinux这个命令么？

```
cmd_link-vmlinux = $(CONFIG_SHELL) $< $(LD) $(LDFLAGS)
```

这$<是make中定义的自动变量，表示依赖关系中的第一个依赖。在vmlinux规则中，第一个依赖是scripts/link-vmlinux.sh。所以接下来的链接工作，就靠它了。

## vmlinux_link

这个脚本中除了链接vmlinux之外，还做了些其他的事情。我们暂且不表。还是来专注看看vmlinux是怎么链接的。

```
info LD vmlinux
vmlinux_link "${kallsymso}" vmlinux
```

然后又调用了vmlinux_link

```
# Link of vmlinux
# ${1} - optional extra .o files
# ${2} - output file
vmlinux_link()
{
	local lds="${objtree}/${KBUILD_LDS}"
	local objects

	if [ "${SRCARCH}" != "um" ]; then
		if [ -n "${CONFIG_THIN_ARCHIVES}" ]; then
			objects="--whole-archive built-in.o ${1}"
		else
			objects="${KBUILD_VMLINUX_INIT}			\
				--start-group				\
				${KBUILD_VMLINUX_MAIN}			\
				--end-group				\
				${1}"
		fi

		${LD} ${LDFLAGS} ${LDFLAGS_vmlinux} -o ${2}		\
			-T ${lds} ${objects}
	else
		if [ -n "${CONFIG_THIN_ARCHIVES}" ]; then
			objects="-Wl,--whole-archive built-in.o ${1}"
		else
			objects="${KBUILD_VMLINUX_INIT}			\
				-Wl,--start-group			\
				${KBUILD_VMLINUX_MAIN}			\
				-Wl,--end-group				\
				${1}"
		fi

		${CC} ${CFLAGS_vmlinux} -o ${2}				\
			-Wl,-T,${lds}					\
			${objects}					\
			-lutil -lrt -lpthread
		rm -f linux
	fi
}
```

截取其中关键的一点

```
${LD} ${LDFLAGS} ${LDFLAGS_vmlinux} -o ${2}		\
	-T ${lds} ${objects}
```

这不就是一个普通的链接么，再看看链接了谁？objects嘛。那objects都是谁呢？就是我们刚才分析的KBUILD_VMLINUX_INIT 和KBUILD_VMLINUX_MAIN呗～

嗯，没错就是它了。


# 一张图总结

最后画一张图吧。


```

                            kernel/built-in.o  certs/built-in.o
                            mm/built-in.o      fs/built-in.o
                            ipc/buit-in.o      security/built-in.o
                            crypto/built-in.o  block/built-in.o
                            lib/built-in.o     lib/lib.a
 head_64.o                  usr/built-in.o     drivers/built-in.o
 head64.o                   sound/built-in.o   firmware/built-in.o
 head.o    init/built-in.o  net/built-in.o     virt/built-in.o
    ||          ||             ||                ||

 $(head-y)    $(init-y)     $(core-y)  $(drivers-y)
                            $(net-y)   $(libs-y)  
                            $(virt-y)
     |            |             |          |
     |            |             |          |
      \          /               \        /
       \        /                 \      /

    KBUILD_VMLINUX_INIT         KBUILD_VMLINUX_MAIN
               \                   /
                 \              /
                       vmlinux

```

嗯，丑是丑了点，不过意思都在了。
