kselftest的源代码在tools/testing/selftests目录下面，不同的测试大类按照目录区分。比如测试cgroup的就在cgroup目录下，测试mm的就在mm目录下。

每个大类都可以单独构建，比如想要测试mm的内容，就可以直接进入到mm目录，运行make。

```
cd linux
cd tools/testing/selftests
cd mm
make
```

下面我们按照自下而上的顺序，先了解如何构建独立的测试用例，再来看如何构建所有的测试用例。

# kselftest_harness/Makefile

我们看selftests中kselftest_harness这个目录下的测试用例，这个是自己测试自己的用例。

文件不多，代码不长。用来学习selftest是很不错的入口。

其中关键的文件是两个

```
Makefile
harness-selftest.c
```

也就是说新增一个测试用例，最少需要两个文件：一个源文件，一个makefile。

源文件的内容我们放到以后来分析，这里主要分析makefile的过程。

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

## 运行测试用例

在测试用例目录下，如tools/testing/selftests/kselftest_harness，执行make run_tests就可以运行当前目录下的测试用例。

这个目标在lib.mk中。

```
define RUN_TESTS
	BASE_DIR="$(selfdir)";			\
	. $(selfdir)/kselftest/runner.sh;	\
	echo $(1);				\
	if [ "X$(summary)" != "X" ]; then       \
		per_test_logging=1;		\
	fi;                                     \
	run_many $(1)
endef

run_tests: all
	@$(call RUN_TESTS, $(TEST_GEN_PROGS) $(TEST_CUSTOM_PROGS) $(TEST_PROGS))
```

当make run_tests执行后，就会运行call RUN_TESTS，而后面的变量都形成一个字符串作为第一个参数传给RUN_TESTS。

最后运行在kselftest/runner.sh脚本中的run_many函数执行测试用例，参数也是那串字符串。

```
run_many()
{
	echo "TAP version 13"
	DIR="${PWD#${BASE_DIR}/}"
	test_num=0
	total=$(echo "$@" | wc -w)
	echo "1..$total"
	for TEST in "$@"; do
		BASENAME_TEST=$(basename $TEST)
		test_num=$(( test_num + 1 ))
		if [ -n "$per_test_logging" ]; then
			logfile="/tmp/$BASENAME_TEST"
			cat /dev/null > "$logfile"
		fi
		if [ -n "$RUN_IN_NETNS" ]; then
			run_in_netns &
		else
			run_one "$DIR" "$TEST" "$test_num"
		fi
	done

	wait
}
```

传入的字符串参数，按照空格切分，有几个就表示有多少测试。然后针对每个测试运行run_one,其中会

  * 读取测试配置文件settings
  * 设置timeout
  * 获取测试命令参数
  * 拼接测试命令行 cmd
  * 运行拼接好的命令行 cmd

其中在执行最终命令时，用了一段有点复杂的脚本：

```
		((((( tap_timeout "$cmd" 2>&1; echo $? >&3) |
			tap_prefix >&4) 3>&1) |
			(read xs; exit $xs)) 4>>"$logfile" &&
		echo "ok $test_num $TEST_HDR_MSG")
```

这里面可以分成几个部分来看

```
  ( tap_timeout "$cmd" 2>&1; echo $? >&3) 
```

执行$cmd，将错误输出合并到标准输出；然后把退出码写道文件描述符3。

```((( ... ) | tap_prefix >&4) 3>&1)
```

将刚才命令的输出，通过tap_prefix格式化，再写到文件描述符4。
然后把刚才写道文件描述符3的退出码再写回标准输出。

```
	((() | (read xs; exit $xs)) 4>>"$logfile" && echo "ok $test_num $TEST_HDR_MSG")
```

把写到标准输出的提出码读到变量xs，然后再退出。
同时将刚才写到文件描述符4的命令运行输出内容，写道日志文件logfile。
如果这一切都顺利，也就是exit $xs也是返回0（表示成功），则会提示测试成功 ok 。

# mm/Makefile

对lib.mk有一定了解后，我们再来找一个例子，看看我们对构建的流程是否真的理解。也巩固一下学到的知识。

文件以include ../lib.mk为界。

前面定义了

  * TEST_GEN_FILES: 这是最终要编译成可执行文件的测试目标
  * TEST_PROGS: 这是make run_tests最后会执行的脚本

之后定义了

  * 目标文件的额外依赖： $(TEST_GEN_FILES): vm_util.c thp_settings.c
  * 链接时额外库： $(OUTPUT)/migration: LDLIBS += -lnuma

这么看，对selftest中构建过程已经有一定了解了。

# Makefile

ok，上面我们了解了单独的一个测试用例是如何构建的，那现在来看看整个selftest的构建过程。这个就要分析tools/testing/selftests/Makefile文件了。

```
# 首先设置了目标，基本就是每个子目录作为一个目标
TARGETS += ...


# 如果在tools/testing/selftests目录下运行make，
# 那么会用下面的方法设置内核源代码的具体对路径
  BUILD := $(CURDIR)
  abs_srctree := $(shell cd $(top_srcdir) && pwd)
  KHDR_INCLUDES := -isystem ${abs_srctree}/usr/include
  DEFAULT_INSTALL_HDR_PATH := 1


all:
	@ret=1;							\
	for TARGET in $(TARGETS) $(INSTALL_DEP_TARGETS); do	\
		BUILD_TARGET=$$BUILD/$$TARGET;			\
		mkdir $$BUILD_TARGET  -p;			\
		$(MAKE) OUTPUT=$$BUILD_TARGET -C $$TARGET	\
				O=$(abs_objtree)		\
				$(if $(FORCE_TARGETS),|| exit);	\
		ret=$$((ret * $$?));				\
	done; exit $$ret;
```

所以这么看，selftests根目录上的Makefile其实没有做什么事。就是设置好了TARGETS后，针对每个target运行make -C $TARGET。

除了all这个目标，还有run_tests,hotpulg等其他目标。不过结构都差不多，就不展开了。
