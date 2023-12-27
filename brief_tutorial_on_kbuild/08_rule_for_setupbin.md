书接上回，bzImage由setup.bin和vmlinux.bin两个文件粘合而成。这次我们来看看setup.bin的诞生记。

# 寻找目标

老套路，第一步就是找一下setup.bin这个目标的规则。还记得之前我们走到哪里了么？对，arch/x86/boot/Makefile。

```
$(obj)/setup.bin: $(obj)/setup.elf FORCE
	$(call if_changed,objcopy)
```

下一步呢？ 对了，找cmd_objcopy。这次定义的地方稍有不同，不在scripts/Makefile.build，而是在scripts/Makefile.lib。

```
# Objcopy
# -------------------------------------

quiet_cmd_objcopy = OBJCOPY $@
cmd_objcopy = $(OBJCOPY) $(OBJCOPYFLAGS) $(OBJCOPYFLAGS_$(@F)) $< $@
```

原来setup.bin是由setup.elf经过objcopy而来。那看来想要弄清楚就要看看setup.elf的来历。走，既然已经到这里了，那咱就再入一层～

# 深入虎穴

```
$(obj)/setup.elf: $(src)/setup.ld $(SETUP_OBJS) FORCE
	$(call if_changed,ld)
```

怎么样，现在是不是驾轻就熟了。看到这个也基本能够猜出个八九不离十。setup.elf是由$(SETUP_OBJS)链接而成的。嗯，没想到这么简单，白白浪费了我这么气势磅礴的一个标题了。

为了弥补点什么，咱把SETUP_OBJS的内容也给大家展开了。

```
SETUP_OBJS = $(addprefix $(obj)/,$(setup-y))
```

原来还套了那么一层:

```
setup-y		+= a20.o bioscall.o cmdline.o copy.o cpu.o cpuflags.o cpucheck.o
setup-y		+= early_serial_console.o edd.o header.o main.o memory.o
setup-y		+= pm.o pmjump.o printf.o regs.o string.o tty.o video.o
setup-y		+= video-mode.o version.o
setup-$(CONFIG_X86_APM_BOOT) += apm.o

# The link order of the video-*.o modules can matter.  In particular,
# video-vga.o *must* be listed first, followed by video-vesa.o.
# Hardware-specific drivers should follow in the order they should be
# probed, and video-bios.o should typically be last.
setup-y		+= video-vga.o
setup-y		+= video-vesa.o
setup-y		+= video-bios.o
```

嗯，够多，终于能勉强配得上咱这个霸气的标题了～

# 图文并茂

setup.bin的编译过程确实简单，来一张图略微总结那么一下子。

```
        a20.o bioscall.o cmdline.o copy.o
        cpu.o cpuflags.o cpucheck.o
        edd.o header.o main.o memory.o
        tty.o pmjump.o printf.o regs.o
        pm.o  string.o video.o
        video-mode.o version.o
        early_serial_console.o

        video-vga.o
        video-vesa.o
        video-bios.o

                    ||
                    \/

				 setup.elf

                    ||
                    \/

				 setup.bin
```

这么一看，东西还挺多的啊。 好了，这个我们也看完啦，是不是感觉so easy~

# 未完待续

bzImage的组成部分除了setup.bin还有另一半—vmlinux.bin。是不是它也这么简单明了呢？是不是它能解除我们心中的一些疑惑呢？

不要走开，即将呈现～
