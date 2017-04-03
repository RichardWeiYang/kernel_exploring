# bzImage isoimage

```
Makefile -- include arch/x86/Makefile
      arch/x86/Makefile
           BOOT_TARGETS = bzlilo bzdisk fdimage fdimage144 fdimage288 isoimage
           bzImage:
	           $(BOOT_TARGETS)
```

#make kernel/fork.o

原来在根目录的Makefile中，单独为.o目标做了这么一个规则。

```
"Makefile"

%.o: %.c prepare scripts FORCE
	$(Q)$(MAKE) $(build)=$(build-dir) $(target-dir)$(notdir $@)
```

#make drivers/net/ethernet/mellanox/mlx4/mlx4_en.ko

原来这个也是在根Makefile目录下有单独的一个目标

```
"Makefile"

%.ko: prepare scripts FORCE
	@echo target is $@
	$(cmd_crmodverdir)
	$(Q)$(MAKE) KBUILD_MODULES=$(if $(CONFIG_MODULES),1)   \
	$(build)=$(build-dir) $(@:.ko=.o)
	$(Q)$(MAKE) -f $(srctree)/scripts/Makefile.modpost
```

#make M=drivers/net/ethernet/mellanox/mlx4/

这个也是在根目录的Makefile中定义

```
"Makefile"

ifeq ("$(origin M)", "command line")
  KBUILD_EXTMOD := $(M)
endif

ifeq ($(KBUILD_EXTMOD),)
_all: all
else
_all: modules
endif

module-dirs := $(addprefix _module_,$(KBUILD_EXTMOD))
PHONY += $(module-dirs) modules
$(module-dirs): crmodverdir $(objtree)/Module.symvers
	$(Q)$(MAKE) $(build)=$(patsubst _module_%,%,$@)

modules: $(module-dirs)
	@$(kecho) '  Building modules, stage 2.';
	$(Q)$(MAKE) -f $(srctree)/scripts/Makefile.modpost
```

首先会解析是不是有M变量在命令行定义。如果有，则添加目标modules。
仔细看通常内核模块编译的两个阶段，在上面这段代码中，就很明显了。

# buit-in.o

在scripts/Makefile.build中定义

```
builtin-target := $(obj)/built-in.o

# If the list of objects to link is empty, just create an empty built-in.o
cmd_link_o_target = $(if $(strip $(obj-y)),\
		      $(cmd_make_builtin) $@ $(filter $(obj-y), $^) \
		      $(cmd_secanalysis),\
		      $(cmd_make_empty_builtin) $@)

$(builtin-target): $(obj-y) FORCE
	$(call if_changed,link_o_target)

```