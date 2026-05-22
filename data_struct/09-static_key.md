我们在内核中常常会看到这样的判断语句：

```
	/* skip numa counters update if numa stats is disabled */
	if (!static_branch_likely(&vm_numa_stat_key))
		return;
```

当配置变化的时侯，又可以用static_branch_enable/static_branch_disable()来改变条件。

我有时候就在想，这个和一般的if判断有什么区别？原来这是内核中的一种优化，可以让代码看上去是动态判断，实际上是静态判断。

接下来我们就来看看这是怎么做到的。

# 定义

首先我们要从定义入手：

```
#define STATIC_KEY_TRUE_INIT  (struct static_key_true) { .key = STATIC_KEY_INIT_TRUE,  }
#define DEFINE_STATIC_KEY_TRUE(name)	\
	struct static_key_true name = STATIC_KEY_TRUE_INIT

DEFINE_STATIC_KEY_TRUE(vm_numa_stat_key);
```

实际上就是定义了一个static_key的结构：

```
struct static_key {
	atomic_t enabled;
#ifdef CONFIG_JUMP_LABEL
/*
 * bit 0 => 1 if key is initially true
 *	    0 if initially false
 * bit 1 => 1 if points to struct static_key_mod
 *	    0 if points to struct jump_entry
 */
	union {
		unsigned long type;
		struct jump_entry *entries;
		struct static_key_mod *next;
	};
#endif	/* CONFIG_JUMP_LABEL */
};
```

这么看上去还是有点复杂。我们一点点展开来看。

# 简化版本

我们在定义中看到有一个#ifdef CONFIG_JUMP_LABEL的条件编译，所以我们先偷懒来看看没有配置这个选项的时候是怎么样的。

## 定义

```
struct static_key {
	atomic_t enabled;
};
```

如果没有配置CONFIG_JUMP_LABEL，static_key就是一个原子变量了。

而当我们定义一个默认为true的static key时，展开如下:

```
#define STATIC_KEY_INIT_TRUE	{ .enabled = ATOMIC_INIT(1) }
#define STATIC_KEY_TRUE_INIT  (struct static_key_true) { .key = STATIC_KEY_INIT_TRUE,  }
#define DEFINE_STATIC_KEY_TRUE(name)	\
	struct static_key_true name = STATIC_KEY_TRUE_INIT

DEFINE_STATIC_KEY_TRUE(vm_numa_stat_key);
```

也就是将enabled的初始值设置为1，这样在后续判断中，就会返回1。

## 改变

```
#define static_branch_enable(x)			static_key_enable(&(x)->key)

static inline void static_key_enable(struct static_key *key)
{
	STATIC_KEY_CHECK_USE(key);

	if (atomic_read(&key->enabled) != 0) {
		WARN_ON_ONCE(atomic_read(&key->enabled) != 1);
		return;
	}
	atomic_set(&key->enabled, 1);
}

static inline void static_key_disable(struct static_key *key)
{
	STATIC_KEY_CHECK_USE(key);

	if (atomic_read(&key->enabled) != 1) {
		WARN_ON_ONCE(atomic_read(&key->enabled) != 0);
		return;
	}
	atomic_set(&key->enabled, 0);
}
```

由此，enable/disable就是设置这个原子变量为1或者0.

## 判断

```
static __always_inline int static_key_count(struct static_key *key)
{
	return raw_atomic_read(&key->enabled);
}

#define static_key_enabled(x)							\
({										\
	if (!__builtin_types_compatible_p(typeof(*x), struct static_key) &&	\
	    !__builtin_types_compatible_p(typeof(*x), struct static_key_true) &&\
	    !__builtin_types_compatible_p(typeof(*x), struct static_key_false))	\
		____wrong_branch_error();					\
	static_key_count((struct static_key *)x) > 0;				\
})

#define static_branch_likely(x)		likely_notrace(static_key_enabled(&(x)->key))
```

这样branch的判断，就变成了原子变量值的判断了。怎么样，这个分支是不是超级简单。

但是如果就是这么实现的话，那和普通的变量判断逻辑就一样了。

好了，偷懒偷过了，该是正经研究代码的时候了。

# 正式版本

现在我来看设置了CONFIG_JUMP_LABEL的版本。

## 定义

```
struct jump_entry {
	s32 code;
	s32 target;
	long key;	// key may be far away from the core kernel under KASLR
};

struct static_key {
	atomic_t enabled;
/*
 * bit 0 => 1 if key is initially true
 *	    0 if initially false
 * bit 1 => 1 if points to struct static_key_mod
 *	    0 if points to struct jump_entry
 */
	union {
		unsigned long type;
		struct jump_entry *entries;
		struct static_key_mod *next;
	};
};
```

我把条件编译的定义去掉了。这么看，static_key除了有一个原子变量外，还有一个联合体。

其中注释上写了这个联合体的含义。恩，又是一个magic的用法。让我们继续往后看。

还是看vm_numa_stat_key这个变量的定义，看看设置了CONFIG_JUMP_LABEL时，定义会有什么变化。

```
#define JUMP_TYPE_TRUE		1UL
#define STATIC_KEY_INIT_TRUE					\
	{ .enabled = ATOMIC_INIT(1),				\
	  .type = JUMP_TYPE_TRUE }
#define STATIC_KEY_TRUE_INIT  (struct static_key_true) { .key = STATIC_KEY_INIT_TRUE,  }
#define DEFINE_STATIC_KEY_TRUE(name)	\
	struct static_key_true name = STATIC_KEY_TRUE_INIT

DEFINE_STATIC_KEY_TRUE(vm_numa_stat_key);
```

这里的差别在与定义时设置了type=JUMP_TYPE_TRUE，也就是为1.

根据注释，意思就是初始化时表示true。

## 改变

还是通过static_branch_enable和static_branch_disable来改变状态。

```
static void jump_label_update(struct static_key *key)
{
	struct jump_entry *stop = __stop___jump_table;
	bool init = system_state < SYSTEM_RUNNING;
	struct jump_entry *entry;

	entry = static_key_entries(key);
	/* if there are no users, entry can be NULL */
	if (entry)
		__jump_label_update(key, entry, stop, init);
}

void static_key_enable_cpuslocked(struct static_key *key)
{
	STATIC_KEY_CHECK_USE(key);
	lockdep_assert_cpus_held();

	if (atomic_read(&key->enabled) > 0) {
		WARN_ON_ONCE(atomic_read(&key->enabled) != 1);
		return;
	}

	jump_label_lock();
	if (atomic_read(&key->enabled) == 0) {
		atomic_set(&key->enabled, -1);
		jump_label_update(key);
		/*
		 * See static_key_slow_inc().
		 */
		atomic_set_release(&key->enabled, 1);
	}
	jump_label_unlock();
}
void static_key_enable(struct static_key *key)
{
	cpus_read_lock();
	static_key_enable_cpuslocked(key);
	cpus_read_unlock();
}
```

所以更改过程中，重要的是jump_label_update函数。（去掉了部分代码。）

接下来，从static_key中得到jump_entry后，就开始工作了。

```
static __always_inline void
__jump_label_transform(struct jump_entry *entry,
		       enum jump_label_type type,
		       int init)
{
	const struct jump_label_patch jlp = __jump_label_patch(entry, type);

	...

	smp_text_poke_single((void *)jump_entry_code(entry), jlp.code, jlp.size, NULL);
}

static void __jump_label_update(struct static_key *key,
				struct jump_entry *entry,
				struct jump_entry *stop,
				bool init)
{
	for (; (entry < stop) && (jump_entry_key(entry) == key); entry++) {
		if (jump_label_can_update(entry, init))
			arch_jump_label_transform(entry, jump_label_type(entry));
	}
}
```

最后在x86上，做工作的是__jump_label_transform()函数。

可以分成两步：

  * __jump_label_patch(): 根据jump_entry中记录的信息生成一个jump指令，或者是nop
  * smp_text_poke_single(): 将jlp这个指令写道jump_entry记录的地址中


## 判断

实际这里没有额外的判断，代码已经被改写。

要么nop什么也不执行，要么就会跳转到已经改写好的目标代码执行。

但是好像还缺了点什么。知道了部分细节后，让我们串起来看一下吧。

# 缺失的细节

虽然原理上，我们从上面的代码已经看到了，但是细节上我们还缺一些东西。

  * jump_entry是谁设置的， 又是怎么链接上static_key的呢？

这是因为刚才偷懒，把判断部分的定义给跳过了。虽然最后逻辑上，判断的过程就是nop或者jmp，但是细节就藏在这里。

## static_branch_likely()不只是判断

通常我们使用static_key时，用的这样的语句：

```
	if (!static_branch_likely(&vm_numa_stat_key))
		return;
```

我们期望的是static_branch_likely()就是一个判断，返回true或者false。但为了能运行时改动逻辑，这里还做了其他。

```
#define JUMP_TABLE_ENTRY(key, label)			\
	".pushsection __jump_table,  \"aw\" \n\t"	\
	_ASM_ALIGN "\n\t"				\
	ANNOTATE_DATA_SPECIAL "\n"			\
	".long 1b - . \n\t"				\
	".long " label " - . \n\t"			\
	_ASM_PTR " " key " - . \n\t"			\
	".popsection \n\t"

#define ARCH_STATIC_BRANCH_ASM(key, label)		\
	"1: .byte " __stringify(BYTES_NOP5) "\n\t"	\
	JUMP_TABLE_ENTRY(key, label)
`
static __always_inline bool arch_static_branch(struct static_key * const key, const bool branch)
{
	asm goto(ARCH_STATIC_BRANCH_ASM("%c0 + %c1", "%l[l_yes]")
		: :  "i" (key), "i" (branch) : : l_yes);

	return false;
l_yes:
	return true;
}

#define static_branch_likely(x)							\
({										\
	bool branch;								\
	if (__builtin_types_compatible_p(typeof(*x), struct static_key_true))	\
		branch = !arch_static_branch(&(x)->key, true);			\
	else if (__builtin_types_compatible_p(typeof(*x), struct static_key_false)) \
		branch = !arch_static_branch_jump(&(x)->key, true);		\
	else									\
		branch = ____wrong_branch_error();				\
	likely_notrace(branch);								\
})
```

如果我们不看arch_static_branch()中的asm goto()，static_branch_likely()返回的就是true。这和我们理解的简单逻辑一致。

那asm goto这段是什么呢？ 就两部分：

  * 一个nop
  * 一个jump_entry

而且关键是这个jump_entry不是生成在当前的代码段，而是通过pushsection放到了__jump_table的代码段。

再来看jump_entry中三个字段分别赋值了什么：

  * code: 1b， 也就是nop指令的地址
  * target: l_yes, 这样跳转到l_yes就会变成返回true，达到目的
  * key: key，也就是当前static_key结构体的地址 和 当前是否要branch(参见jump_entry_is_branch())

## __jump_table的家

刚才说到，每个jump_entry都定义在__jump_table段中，那我们来看看这个段定义的范围。

```
#define BOUNDED_SECTION_PRE_LABEL(_sec_, _label_, _BEGIN_, _END_)	\
	_BEGIN_##_label_ = .;						\
	KEEP(*(_sec_))							\
	_END_##_label_ = .;

#define BOUNDED_SECTION_BY(_sec_, _label_)				\
	BOUNDED_SECTION_PRE_LABEL(_sec_, _label_, __start, __stop)

#define JUMP_TABLE_DATA							\
	. = ALIGN(8);							\
	BOUNDED_SECTION_BY(__jump_table, ___jump_table)
```

这里定义了一个"__jump_table"的section，头尾的边界是__start___jump_table和__stop___jump_table。

还记得这个__stop___jump_table吗？上面好像看到过。

## static_key和jump_entry的关联

我们在jump_entry的定义中看到了它指向了static_key，但是我们在static_key的定义中，并没有看到它关联到了jump_entry。

  * DEFINE_STATIC_KEY_TRUE()仅仅设置了.type，对应的entries是空的
  * 在jump_label_update()更新时，又要从static_key_entries()中找出所有的entry

所以static_key中的entries是在jump_label_init()中绑定的。

所以在运行的时候，一个static_key长这样：

```
     static_key                 __start___jump_table
     +--------------+           +---------+
     |              |           |         |
     |entries       |---------->|         |
     |              |           |         |
     +--------------+           |         |
                                |         |
                                |         |
                                +---------+
```

从entries指向的位值开始，所有key是static_key的jump_entry都是使用了这个key的代码点。
