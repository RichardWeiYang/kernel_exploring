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
