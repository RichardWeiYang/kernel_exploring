我们看selftests中kselftest_harness这个目录下的测试用例，这个是自己测试自己的用例。

文件不多，代码不长。用来学习selftest是很不错的入口。

其中关键的文件是两个

```
Makefile
harness-selftest.c
```

也就是说新增一个测试用例，最少需要两个文件：一个源文件，一个makefile。

源文件的内容我们放到以后来分析，这里主要分析makefile的过程。

# Makefile

打开这个makefile，发现内容非常少。但是最后又包含了lib.mk。所以重要的工作实际放在了这个库中。

```
TEST_GEN_PROGS_EXTENDED := harness-selftest
TEST_PROGS := harness-selftest.sh
EXTRA_CLEAN := harness-selftest.seen

include ../lib.mk
```

# lib.mk

从上面makefile的写法来看，说明lib.mk中定义了几个变量名。将这些变量的内容设定后，就可以利用lib.mk的规则生成想要的目标。

现在我们就打开这个lib.mk文件看看，这些是如何工作的。为我们后续的工作做准备。以下是对lib.mk文件的总体分析：

```
ifneq ($(LLVM),)
...
else
CC := $(CROSS_COMPILE)gcc    # 先设置了要使用的编译器
endif # LLVM

# 如果没有指定的话，当前目录就是目标输出的目录。且是绝对路径
OUTPUT := $(shell pwd)
# 将selfdir设置为lib.mk所在的绝对路径
selfdir = $(realpath $(dir $(filter %/lib.mk,$(MAKEFILE_LIST))))
# 将top_srcdir设置为linux内核源码根目录
top_srcdir = $(selfdir)/../../..


# 这里已经看到了makefile中的一个变量TEST_GEN_PROGS_EXTENDED 
# 就是将目标加上输出目录的路径
TEST_GEN_PROGS := $(patsubst %,$(OUTPUT)/%,$(TEST_GEN_PROGS))
TEST_GEN_PROGS_EXTENDED := $(patsubst %,$(OUTPUT)/%,$(TEST_GEN_PROGS_EXTENDED))
TEST_GEN_FILES := $(patsubst %,$(OUTPUT)/%,$(TEST_GEN_FILES))


# 这里是整个构建过程的目标，可以看到主要就是上面三个变量指定的
all: $(TEST_GEN_PROGS) $(TEST_GEN_PROGS_EXTENDED) $(TEST_GEN_FILES) \
	$(if $(TEST_GEN_MODS_DIR),gen_mods_dir)


# 接下来文件中定义了一些target，如install, run_tests。这些我们先跳过
...

# 构建OUTPUT的规则, 使用者还能定义OVERRIDE_TARGETS来覆盖规则
ifeq ($(OVERRIDE_TARGETS),)
LOCAL_HDRS += $(selfdir)/kselftest_harness.h $(selfdir)/kselftest.h
$(OUTPUT)/%:%.c $(LOCAL_HDRS)
	$(call msg,CC,,$@)
	$(Q)$(LINK.c) $(filter-out $(LOCAL_HDRS),$^) $(LDLIBS) -o $@

$(OUTPUT)/%.o:%.S
	$(COMPILE.S) $^ -o $@

$(OUTPUT)/%:%.S
	$(LINK.S) $^ $(LDLIBS) -o $@
endif
```

## 显示出构建具体命令

在lib.mk中，有一段定义了Q/msg命令。通常来说，构建过程会隐藏构建目标时具体使用的命令。

但是有时候我们需要知道构建时究竟用了什么参数，包含头文件的路径。所以希望能展示出详细的命令行。

根据lib.mk中的定义，我们执行时加上V=1就可以了。如

```
$make V=1
gcc -D_GNU_SOURCE=     harness-selftest.c  -o /home/richard/git/linux/tools/testing/selftests/kselftest_harness/harness-selftest
```
