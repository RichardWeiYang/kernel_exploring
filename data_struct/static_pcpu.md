内核中通常会定义很多的pcpu变量，这样有几个好处

* 增加数据访问的并发量
* 减少数据访问的时延

从定义上就可以看出pcpu变量就是每个cpu都有某个变量的副本，各自访问各自的。那在实现上是怎么做的呢？我们今天就来看一下。

# 如何定义

我们先来看静态pcpu变量是如何定义的。

通常我们定义一个pcpu变量使用这样的语句。

```
DEFINE_PER_CPU(int, numa_node);
```

这样就定义了一个int类型，名字为numa_node的变量。接下来就深入研究一下。

## DEFINE_PER_CPU

```
#define DEFINE_PER_CPU_SECTION(type, name, sec)				\
	__PCPU_ATTRS(sec) __typeof__(type) name
#endif

#define DEFINE_PER_CPU(type, name)					\
	DEFINE_PER_CPU_SECTION(type, name, "")
```

看最后一行，最终也就是定义了一个type类型，名字是name的变量。感觉和普通的变量没有什么区别。那区别在哪里呢？对了，就在__PCPU_ATTRS宏里面。

## __PCPU_ATTRS

```
#define __PCPU_ATTRS(sec)						\
	__percpu __attribute__((section(PER_CPU_BASE_SECTION sec)))	\
	PER_CPU_ATTRIBUTES

#define PER_CPU_BASE_SECTION ".data..percpu"
```

好了，这个比较明确了，就是给定义的变量添加了一个section的修饰符。section是什么概念呢？你可以理解为同一个section的变量会放在同一块存储区。一会儿我们再来看这个概念。

## 展开后

```
DEFINE_PER_CPU(int, numa_node);

=

__attribute__((section(".data..percpu")))  \
    __typeof__(int) numa_node;
```

这样看或许能够清楚一些。

pcpu变量和普通变量定义时的差别在于pcpu变量被安排在了一个指定的section中。

PS： 可以用make xxx.i将某个使用DEFINE_PER_CPU()定义了变量的文件展开，就能验证了。

## 放在哪

已经看到变量定义在某一个section了，但是还是不死心，想要看看究竟是怎么放的。

好吧，我就带你来看看。

首先在include/asm-generic/vmlinux.lds.h中定义了PERCPU_INPUT。

```
#define PERCPU_INPUT(cacheline)						\
	__per_cpu_start = .;						\
	. = ALIGN(PAGE_SIZE);						\
	*(.data..percpu..page_aligned)					\
	. = ALIGN(cacheline);						\
	__per_cpu_hot_start = .;					\
	*(SORT_BY_ALIGNMENT(.data..percpu..hot.*))			\
	__per_cpu_hot_end = .;						\
	. = ALIGN(cacheline);						\
	*(.data..percpu..read_mostly)					\
	. = ALIGN(cacheline);						\
	*(.data..percpu)						\
	*(.data..percpu..shared_aligned)				\
	PERCPU_DECRYPTED_SECTION					\
	__per_cpu_end = .;
```

你看，凡是.data..percpu开头的都包含在这个定义内了。

在同一个文件中又定义了一个包含这个定义的定义PERCPU_SECTION。

```
#define PERCPU_SECTION(cacheline)					\
	. = ALIGN(PAGE_SIZE);						\
	.data..percpu	: AT(ADDR(.data..percpu) - LOAD_OFFSET) {	\
		PERCPU_INPUT(cacheline)					\
	}
```

而这个PERCPU_SECTION定义最终包含在文件arch/x86/kernel/vmlinux.lds.S中。这就是我们最后链接vmlinux时使用的脚本。

这样通过DEFINE_PER_CPU()定义的变量，就在链接的时候有了固定的位置。（所以这算全局变量。）

# 如何访问

定义看完了，来看看要怎么访问。

还是看刚才的numa_node变量，我们要访问某cpu上的变量时通过如下的语句。

```
per_cpu(numa_node, cpu);
```

好了，那来看看都是些什么吧

## per_cpu()

```
#define per_cpu(var, cpu)	(*per_cpu_ptr(&(var), cpu))
```

```
#define per_cpu_ptr(ptr, cpu)						\
({									\
	__verify_pcpu_ptr(ptr);						\
	SHIFT_PERCPU_PTR((ptr), per_cpu_offset((cpu)));			\
})
```

卧槽，这么长。不着急，仔细看其实有的不用。比如这个__verif_pcpu_ptr()，一看就是用来检测了，就先跳过吧。关键是这个SHIFT_PERCPU_PTR。

## per_cpu_offset()

先从软柿子开始捏，per_cpu_offset()就一个数组。

```
extern unsigned long __per_cpu_offset[NR_CPUS];

#define per_cpu_offset(x) (__per_cpu_offset[x])
```

然后就没这么简单了，好像是系统在初始化的时候会设置这个数组的值。不过好在原理比较简单，就是给每一个cpu划分了一个区域，区域的起始地址就保存在这个数组里。

## SHIFT_PERCPU_PTR()

```
/*
 * Add an offset to a pointer.  Use RELOC_HIDE() to prevent the compiler
 * from making incorrect assumptions about the pointer value.
 */
#define SHIFT_PERCPU_PTR(__p, __offset)					\
	RELOC_HIDE(PERCPU_PTR(__p), (__offset))
```

RELOC_HIDE:分别传入了变量的一个指针，和刚才我们看到的offset。

得，继续。

## PERCPU_PTR()

```
#define PERCPU_PTR(__p)							\
	(TYPEOF_UNQUAL(*(__p)) __force __kernel *)((__force unsigned long)(__p))
```

感觉就是定义了__p类型的变量？没懂，先这样吧。

## RELOC_HIDE()

```
#define RELOC_HIDE(ptr, off)					\
  ({ unsigned long __ptr;					\
     __ptr = (unsigned long) (ptr);				\
    (typeof(ptr)) (__ptr + (off)); })
```

又是一个很长，但是其实很简单的定义。是什么呢？ 你看最后一行，其实就是一个指针加上了offset。

## 展开后

```
per_cpu(numa_node, cpu)

=

*(&numa_node + per_cpu_offset(cpu))
```

好了，这样看可能可以清楚一些。

但是呢，还是有点不清楚，让我再来把实现的细节讲一下。这样，我估计你就可以彻底理解了。

# 背后的实现

先透露一下，其实pcpu变量的实现就是给每个cpu都分配一块内存，并且记录下每块区域和原生区域之间的offset。所以访问的时候就是直接通过原生变量的地址加上目标cpu变量区域的offset就可以了。

貌似有点饶，来看一张图吧。

```
 __per_cpu_start +-------------------+  -+-   -+-
                 |                   |   |     |
                 |                   |   |     |
                 |                   |   |     |
 __per_cpu_end   +-------------------+   |     |
                                         |     |
                                         |     |
                                         |     |
                                               |
                 Group0                        |
  pcpu_base_addr +-------------------+   per_cpu_offset(2)
                 |cpu0               |         |    
                 |                   |   |     |    
                 |                   |   |     |
                 +-------------------+   |     |
                 |cpu1               |   |     |
                 |                   |   |      
                 |                   |   |     per_cpu_offset(3)                        
                 +-------------------+  -+-                             
                 |cpu2               |         |
                 |                   |         |
                 |                   |         |
                 +-------------------+         |
                                               |
                                               |
                                               |
                                               |
                 Group1                        |
                 +-------------------+        -+-
                 |cpu3               |
                 |                   |
                 |                   |
                 +-------------------+
                 |cpu4               |
                 |                   |
                 |                   |
                 +-------------------+ -+-
                 |cpu5               |  |
                 |                   |  pcpu_unit_size
                 |                   |  |
                 +-------------------+ -+-
```

其中__per_cpu_start和__per_cpu_end就是刚才我们看到的那个section的定义。所有pcpu变量都会被保存在这个地址空间内。

在系统启动的时候，会给每一个cpu分配各自的pcpu变量内存空间。并计算出每个区域相对于__per_cpu_start的offset。

具体的代码在setup_per_cpu_areas()函数中，有兴趣的可以再深入研究。具体实现就不在这里展开了。
