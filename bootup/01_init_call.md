
在内核代码中经常会看到core_initcall(), subsys_initcall()这样xxx_initcall()的函数。

这些函数可以理解为c++中的构造函数，只是内核对这些函数做了分类，并且在特定的地方调用他们。

今天我们就来学习一下。

# 函数定义

先来看看都有哪些类似的函数：

```
#define early_initcall(fn)		__define_initcall(fn, early)

/*
 * A "pure" initcall has no dependencies on anything else, and purely
 * initializes variables that couldn't be statically initialized.
 *
 * This only exists for built-in code, not for modules.
 * Keep main.c:initcall_level_names[] in sync.
 */
#define pure_initcall(fn)		__define_initcall(fn, 0)

#define core_initcall(fn)		__define_initcall(fn, 1)
#define core_initcall_sync(fn)		__define_initcall(fn, 1s)
#define postcore_initcall(fn)		__define_initcall(fn, 2)
#define postcore_initcall_sync(fn)	__define_initcall(fn, 2s)
#define arch_initcall(fn)		__define_initcall(fn, 3)
#define arch_initcall_sync(fn)		__define_initcall(fn, 3s)
#define subsys_initcall(fn)		__define_initcall(fn, 4)
#define subsys_initcall_sync(fn)	__define_initcall(fn, 4s)
#define fs_initcall(fn)			__define_initcall(fn, 5)
#define fs_initcall_sync(fn)		__define_initcall(fn, 5s)
#define rootfs_initcall(fn)		__define_initcall(fn, rootfs)
#define device_initcall(fn)		__define_initcall(fn, 6)
#define device_initcall_sync(fn)	__define_initcall(fn, 6s)
#define late_initcall(fn)		__define_initcall(fn, 7)
#define late_initcall_sync(fn)		__define_initcall(fn, 7s)
```

这些函数不仅有相似的名字，还有这相似的定义：

```
#define ___define_initcall(fn, id, __sec) \
	static initcall_t __initcall_##fn##id __used \
		__attribute__((__section__(#__sec ".init"))) = fn;

#define __define_initcall(fn, id) ___define_initcall(fn, id, .initcall##id)
```

可以看到这些函数地址都保存在类型为initcall_t的变量中。而这个变量的定义上又加了对应的section属性。

那这个section的名字展开是什么呢？以early_initcall()为例，这个section展开后是 .initcallearly.init

# 函数链接

既然看到了section定义，那这个东西就要和链接脚本联系起来。

在x86上使用的脚本是arch/x86/kernel/vmlinux.lds.S，其中简化后长这样：

```
SECTIONS
{
  ...
  INIT_DATA_SECTION(16)
  ...
}
```

而INIT_DATA_SECTION的定义在文件include/asm-generic/vmlinux.lds.h中：

```
#define INIT_CALLS_LEVEL(level)						\
		__initcall##level##_start = .;				\
		KEEP(*(.initcall##level##.init))			\
		KEEP(*(.initcall##level##s.init))			\

#define INIT_CALLS							\
		__initcall_start = .;					\
		KEEP(*(.initcallearly.init))				\
		INIT_CALLS_LEVEL(0)					\
		INIT_CALLS_LEVEL(1)					\
		INIT_CALLS_LEVEL(2)					\
		INIT_CALLS_LEVEL(3)					\
		INIT_CALLS_LEVEL(4)					\
		INIT_CALLS_LEVEL(5)					\
		INIT_CALLS_LEVEL(rootfs)				\
		INIT_CALLS_LEVEL(6)					\
		INIT_CALLS_LEVEL(7)					\
		__initcall_end = .;

#define INIT_DATA_SECTION(initsetup_align)				\
	.init.data : AT(ADDR(.init.data) - LOAD_OFFSET) {		\
		INIT_DATA						\
		INIT_SETUP(initsetup_align)				\
		INIT_CALLS						\
		CON_INITCALL						\
		INIT_RAM_FS						\
	}
```

所以源代码中定义的所有函数都有各自对应的一个section，而且这么多函数都放在__initcall_start和__initcall_end指定的一段空间。

# 函数调用

好了，找了这么半天都还没有看到这些init函数究竟是在哪里被调用的。这又是一堆很狗血的东西，要不然为啥每次找都要花上一段时间呢。

还是直接写出调用的点吧，好像也没有什么可以多说的。

```
start_kernel()
  ...
  rest_init()              <--- almost last step in start_kernel()
    kernel_init()
      ...
      kernel_init_freeable()
        do_pre_smp_initcalls()
          for (fn = __initcall_start; fn < __initcall0_start; fn++)
            do_one_initcall(initcall_from_entry(fn));
        ...
        do_basic_setup()
          do_initcalls()
            for (level = 0; level < ARRAY_SIZE(initcall_levels) - 1; level++)
              do_initcall_level(level, command_line)
```

节奏上一共分成两个部分，第一部分只调用了__initcall_start到__initcall0_start之间的初始化函数。

而第二部分则是调用剩下的，要证实这点还要看initcall_levels的定义。

```
static initcall_entry_t *initcall_levels[] __initdata = {
	__initcall0_start,
	__initcall1_start,
	__initcall2_start,
	__initcall3_start,
	__initcall4_start,
	__initcall5_start,
	__initcall6_start,
	__initcall7_start,
	__initcall_end,
};
```

因为每一个level中包含多个定义好的函数，所以在do_initcall_level中还有一个循环调用每一个具体的函数。
