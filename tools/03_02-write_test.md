这里我们来看下如何编写selftest。还是从简单的例子开始，在目录kselftest_harness中的代码就是很好的开始。

这个例子使用了新版本的测试框架，引用的是kselftest_harness.h头文件。老版本的引用的是kselftest.h。

从kselftest_harness.h文件的文档中，我们可以看到使用方法。结构比较清晰：

  * TEST/TEST_F来定义测试用例，写一个就表示一个用例
  * TEST定义的是比较简单的，没有额外的数据
  * TEST_F定义的相对复杂，可以用FIXTURE定义一个数据结构，并且在测试正式运行前后，通过FIXTURE_SETUP/FIXTURE_TEARDOWN来准备和销毁测试数据。
  * 用ASSERT_EQ等宏来判断测试是否成功
  * 用SKIP来上报跳过测试，如系统配置不符合等情况

# 定义测试用例的两种方式

例子中我们可以看到有两种定义测试用例的方式，TEST()简洁一些，TEST_F()有更多的定制空间。

## TEST()

```
#define TEST(test_name) __TEST_IMPL(test_name, -1)

#define __TEST_IMPL(test_name, _signal) \
	static void test_name(struct __test_metadata *_metadata); \
	static void wrapper_##test_name( \
		struct __test_metadata *_metadata, \
		struct __fixture_variant_metadata __attribute__((unused)) *variant) \
	{ \
		test_name(_metadata); \
	} \
	static struct __test_metadata _##test_name##_object = \
		{ .name = #test_name, \
		  .fn = &wrapper_##test_name, \
		  .fixture = &_fixture_global, \
		  .termsig = _signal, \
		  .timeout = TEST_TIMEOUT_DEFAULT, }; \
	static void __attribute__((constructor)) _register_##test_name(void) \
	{ \
		__register_test(&_##test_name##_object); \
	} \
	static void test_name( \
		struct __test_metadata __attribute__((unused)) *_metadata)
```

这个定义还是比较简洁的:

  * 定义了一个__test_metadata结构的变量(_##test_name##_object), 这个变量中包含了定义的代码块,预期的信号量,超时设置
  * 通过构造函数_register_##test_name，将变量注册到某个链表上

## TEST_F()

```
#define TEST_F(fixture_name, test_name) \
	__TEST_F_IMPL(fixture_name, test_name, -1, TEST_TIMEOUT_DEFAULT)


#define __TEST_F_IMPL(fixture_name, test_name, signal, tmout) \
	static void fixture_name##_##test_name( \
		struct __test_metadata *_metadata, \
		FIXTURE_DATA(fixture_name) *self, \
		const FIXTURE_VARIANT(fixture_name) *variant); \
	static void wrapper_##fixture_name##_##test_name( \
		struct __test_metadata *_metadata, \
		struct __fixture_variant_metadata *variant) \
	{ \
	...
	...
	...
	} \
	static void wrapper_##fixture_name##_##test_name##_teardown( \
		bool in_parent, struct __test_metadata *_metadata, \
		void *self, const void *variant) \
	{ \
		if (fixture_name##_teardown_parent == in_parent && \
				!__atomic_test_and_set(_metadata->no_teardown, __ATOMIC_RELAXED)) \
			fixture_name##_teardown(_metadata, self, variant); \
	} \
	static struct __test_metadata *_##fixture_name##_##test_name##_object; \
	static void __attribute__((constructor)) \
			_register_##fixture_name##_##test_name(void) \
	{ \
		struct __test_metadata *object = mmap(NULL, sizeof(*object), \
			PROT_READ | PROT_WRITE, MAP_SHARED | MAP_ANONYMOUS, -1, 0); \
		object->name = #test_name; \
		object->fn = &wrapper_##fixture_name##_##test_name; \
		object->fixture = &_##fixture_name##_fixture_object; \
		object->teardown_fn = &wrapper_##fixture_name##_##test_name##_teardown; \
		object->termsig = signal; \
		object->timeout = tmout; \
		_##fixture_name##_##test_name##_object = object; \
		__register_test(object); \
	} \
	static void fixture_name##_##test_name( \
		struct __test_metadata __attribute__((unused)) *_metadata, \
		FIXTURE_DATA(fixture_name) __attribute__((unused)) *self, \
		const FIXTURE_VARIANT(fixture_name) \
			__attribute__((unused)) *variant)
```

相比前面一个定义，这个要长很多。为了看清楚，我先把中间一个wrapper函数的定义省略。

这么来看，主要工作和上面差不多：

  * 定义了一个__test_metadata结构的变量
  * 定义了一个构造函数，将变量注册到链表上

区别是这次的变量，是通过mmap(MAP_SHARED)分配的。PS: 其实我不太懂，全局变量不应该也是大家都能访问的吗？

接下来我们就看看，刚才省略的，被写入object->fn的定义wrapper_##fixture_name##_##test_name。

# 测试的启动

当我们用kselftest_harness.h时，最后只要写上TEST_HARNESS_MAIN就行了。打开一看，实际上是调用了test_harness_run()。

那就让我们来仔细研究一下这个函数test_harness_run()。

```

ret = test_harness_argv_check(argc, argv);


for (f = __fixture_list; f; f = f->next) {
	for (v = f->variant ?: &no_variant; v; v = v->next) {
		unsigned int old_tests = test_count;

		for (t = f->tests; t; t = t->next)
			if (test_enabled(argc, argv, f, v, t))
				test_count++;

		if (old_tests != test_count)
			case_count++;

	}
}

ksft_print_header();
ksft_set_plan(test_count);
ksft_print_msg("Starting %u tests from %u test cases.\n",
       test_count, case_count);


for (f = __fixture_list; f; f = f->next) {
	for (v = f->variant ?: &no_variant; v; v = v->next) {
		for (t = f->tests; t; t = t->next) {
			if (!test_enabled(argc, argv, f, v, t))
				continue;
			...
			__run_test(f, v, t);
			t->results = NULL;
			if (__test_passed(t))
				pass_count++;
			else
				ret = 1;
		}
	}
}
```

从结构上来说，其实很清晰。就做了两件事情：

  * 数一数一共有多少测试用例
  * 运行测试，并记录测试结果

那我们就一个个展开吧。

## 帮助信息

看了test_harness_run()才知道，原来测试用例还有帮助。赶紧运行一下，看看都是啥。

```
$./harness-selftest -h
Usage: ./harness-selftest [-h|-l] [-t|-T|-v|-V|-f|-F|-r name]
	-h       print help
	-l       list all tests

	-t name  include test
	-T name  exclude test
	-v name  include variant
	-V name  exclude variant
	-f name  include fixture
	-F name  exclude fixture
	-r name  run specified test

Test filter options can be specified multiple times. The filtering stops
at the first match. For example to include all tests from variant 'bla'
but not test 'foo' specify '-T foo -v bla'.
```

其中-l这个选项很有意思，再执行一下看看。

```
$./harness-selftest -l
# FIXTURE            VARIANT                   TEST
global                                         standalone_pass
                                               standalone_fail
                                               signal_pass
                                               signal_fail
--------------------------------------------------------------------------------
fixture                                        pass
                                               fail
                                               timeout
--------------------------------------------------------------------------------
fixture_parent                                 pass
--------------------------------------------------------------------------------
fixture_setup_failure                           pass
```

从这个输出中，我们可以看到kselftest的测试用例有三个层次： 

  * fixture
  * variant
  * test

运行时我们还可以通过参数-t/-f等来设定需要运行哪些测试用例。

## 选中测试用例

```
for (f = __fixture_list; f; f = f->next) {
	for (v = f->variant ?: &no_variant; v; v = v->next) {
		unsigned int old_tests = test_count;

		for (t = f->tests; t; t = t->next)
			if (test_enabled(argc, argv, f, v, t))
				test_count++;

		if (old_tests != test_count)
			case_count++;
	}
}
```

这个三层循环很明显告诉我们测试用例的三层结构：fixture/variant/test。

所以test是最小的粒度，一共运行了多少测试用例，说的就是这个数字。

其中test_enabled()用来比较命令行上传入的名字和f/v/t中的名字是否匹配。

## 运行测试用例

运行的整理过程和上面差不多，也是按照f/v/t的顺序。对需要运行的测试，则使用__run_test()

__run_test()的运行过程也不难，他会fork一个子进程来调用t->fn()，把返回的状态写到t->exit_code中。由__test_passed()来判断测试用例是否通过测试。

这样的话，问题就又回到了t->fn。这个就是在TEST()/TEST_F()中设置的值。

对TEST()来说，这个函数就是用户定义的样子，不过是套了一个壳子。但对TEST_F()来说就有点不一样了。因为这个套的壳子有点大: wrapper_##fixture_name##_##test_name

我们摘出比较关键的部分看一下：

```
static void wrapper_##fixture_name##_##test_name( \
	struct __test_metadata *_metadata, \
	struct __fixture_variant_metadata *variant) \
{ \
	...

	/* Makes sure there is only one teardown, even when child forks again. */ \
	_metadata->no_teardown = mmap(NULL, sizeof(*_metadata->no_teardown), \
		PROT_READ | PROT_WRITE, MAP_SHARED | MAP_ANONYMOUS, -1, 0); \
	*_metadata->no_teardown = true; \

	...

	_metadata->variant = variant->data; \
	_metadata->self = self; \
	child = fork(); \
	if (child == 0) { \
		fixture_name##_setup(_metadata, self, variant->data); \
		/* Let setup failure terminate early. */ \
		if (_metadata->exit_code) \
			_exit(0); \
		*_metadata->no_teardown = false; \
		fixture_name##_##test_name(_metadata, self, variant->data); \
		_metadata->teardown_fn(false, _metadata, self, variant->data); \
		_exit(0); \
	} else if (child < 0 || child != waitpid(child, &status, 0)) { \
		ksft_print_msg("ERROR SPAWNING TEST GRANDCHILD\n"); \
		_metadata->exit_code = KSFT_FAIL; \
	} \
	_metadata->teardown_fn(true, _metadata, self, variant->data); \
	munmap(_metadata->no_teardown, sizeof(*_metadata->no_teardown)); \
	_metadata->no_teardown = NULL; \

```

还是fork了一个子进程来完成测试。进入子进程后，先setup，然后运行测试，最后teardown。

有意思的是teardown_fn在父子进程中，各运行了一次，只是第一个参数不同，表示了是在父进程还是在子进程执行清理工作。

这个和如何定义teardown有关：

  * FIXTURE_TEARDOWN        : 在子进程中清理
  * FIXTURE_TEARDOWN_PARENT : 在父进程中清理

## 增加variant

在__run_test()中我们已经看到for循环中第二层是遍历fixture->variant链表。在之前的例子中我们都没有看到过，这里我们看一下是如何使用的。

我们找到了有使用variant的例子代码。

```
FIXTURE_VARIANT(access)
{
	const bool mount_exec;
	const bool file_exec;
};

/* clang-format off */
FIXTURE_VARIANT_ADD(access, mount_exec_file_exec) {
	/* clang-format on */
	.mount_exec = true,
	.file_exec = true,
};
```

在这里，首先用FIXTURE_VARIANT定义一个你想要的数据结构，然后通过FIXTURE_VARIANT_ADD将这个结构添加到fixture->variant链表上。

```
test_harness_run:
	for (f = __fixture_list; f; f = f->next) {
		for (v = f->variant ?: &no_variant; v; v = v->next) {
			for (t = f->tests; t; t = t->next) {
				...
				__run_test(f, v, t);
				...
			}
		}
	}

__run_test:
	t->fn(t, variant);

wrapper_##fixture_name##_##test_name:
	fixture_name##_setup(_metadata, self, variant->data); \
	fixture_name##_##test_name(_metadata, self, variant->data); \
	_metadata->teardown_fn(false, _metadata, self, variant->data); \
```

通过一系列的调用传递，最后把variant->data传给实际运行的函数。而这个variant->data就是用FIXTURE_VARIANT定义的结构。

其中传递了variant->data的三个函数分别是：

  * fixture_name##_setup: FIXTURE_SETUP()定义的代码块
  * fixture_name##_##test_name: TEST_F()定义的代码块
  * teardown_fn: 基本可以认为是FIXTURE_TEARDOWN[_PARENT]定义的代码块
