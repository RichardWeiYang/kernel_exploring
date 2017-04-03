通常的makefile调试手段学会了后，个人使用了一个调试内核编译过程的小窍门。

因为内核的文件超多，各个makefile又是大家通用的。所以如果在某个makefile中添加打印信息，那估计就是满眼输出，你都不知道看哪里了。

#思路

那可以怎么做呢？

我们不是已经看到内核编译的整体架构是

```
build := -f $(srctree)/scripts/Makefile.build obj
	$(Q)$(MAKE) $(build)=$@
```

好了，我想到的方法就是，在Make某个builtin.o的时候，强行把规则改了，变成使用另一个Makefile.build。这样就可以在自己添加的Makefile.build中，添加调试输出了。

#对net/9p/目录调试的例子
下面给个例子，拿net/9p/这个目录举例。

比如我们可以

> make net/9p/built-in.o

来编译这个文件。但是可能在编译的时候发现没有达到我们的预期，或者是想看看这个编译的过程都是什么样子的，学习一下kbuild的结构。那么可以做如下改动。

```
diff --git a/Makefile b/Makefile
index 2203500..dac1344 100644
--- a/Makefile
+++ b/Makefile
@@ -1547,6 +1547,8 @@ endif
        $(Q)$(MAKE) $(build)=$(build-dir) $(target-dir)$(notdir $@)
+net/9p/built-in.o: prepare scripts
+       $(Q)$(MAKE) -f $(srctree)/scripts/Makefile.build_9p obj=$(build-dir) $(target-dir)$(notdir $@)
 
 # Modules
 /: prepare scripts FORCE
diff --git a/scripts/Makefile.build_9p b/scripts/Makefile.build_9p
index 2c47f9c..cce85c6 100644
--- a/scripts/Makefile.build_9p
+++ b/scripts/Makefile.build_9p
@@ -1,6 +1,7 @@
 # ==========================================================================
 # Building
 # ==========================================================================
+$(warning $(obj))
 
 src := $(obj)

@@ -43,6 +44,9 @@ kbuild-dir := $(if $(filter /%,$(src)),$(src),$(srctree)/$(src))
 kbuild-file := $(if $(wildcard $(kbuild-dir)/Kbuild),$(kbuild-dir)/Kbuild,$(kbuild-dir)/Makefile)
 include $(kbuild-file)
 
+$(warning "obj-y " $(obj-y))
+$(warning "obj-m " $(obj-m))
+
 # If the save-* variables changed error out
 ifeq ($(KBUILD_NOPEDANTIC),)
         ifneq ("$(save-cflags)","$(CFLAGS)")
```

注： 这个里面 其实我拷贝了一份scripts/Makefile.build 到 scripts/Makefile.build_9p。为了突出重点，这里只显示添加的调试信息。

这样修改后， make net/9p/built-in.o 的输出如下：

```
:~/git/linux$ make net/9p/built-in.o 
  CHK     include/config/kernel.release
  CHK     include/generated/uapi/linux/version.h
  CHK     include/generated/utsrelease.h
  CHK     include/generated/bounds.h
  CHK     include/generated/timeconst.h
  CHK     include/generated/asm-offsets.h
  CALL    scripts/checksyscalls.sh
scripts/Makefile.build_9p:4: net/9p
scripts/Makefile.build_9p:47: "obj-y " 
scripts/Makefile.build_9p:48: "obj-m " 9pnet.o 9pnet_virtio.o 9pnet_rdma.o
make[1]: 'net/9p/built-in.o' is up to date.
```

是不是看到了常见的 obj-y, obj-m的定义了？

好了，其他的调试招数就都可以用了哦～