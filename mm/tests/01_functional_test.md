在内核中为了保证代码质量，有很多功能测试的测试用例。在改完代码后运行一下，可以初步检查出问题。

这里列举几个和mm相关的测试。一般都在tools/testing目录下。

# 基础数据结构

内核中有很多基础数据结构如radix_tree/maple_tree/rbtree等。这些关键的数据结构也有测试用例。

他们的测试目录分别是

  * tools/testing/radix-tree
  * tools/testing/radix-tree
  * tools/testing/rbtree

使用方法同上。

# memblock

测试代码在tools/testing/memblock目录下。

进入该目录后直接make，生成测试程序，然后直接运行./main。

也可以 make NUMA=1，测试有numa的情况。

# vma

测试代码在tools/testing/vma目录下。

使用方法同上。

# selftests

测试代码在tools/testing/selftests/mm目录下。

进入该目录后直接make，生成测试程序。

该目录下有个执行脚本run_vmtests.h，通过-h可以查看如何运行测试程序。

```
$./run_vmtests.sh -h
```

默认情况下，什么都不加会把所有测试都跑一遍。一般会通过-t指定类型，如mmap就测试和mmap相关的。

具体在脚本中是通过"CATEGORY="hugetlb" run_test ./map_hugetlb"这样的命令来最后运行实际的测试程序。

在run_test函数中，判断CATEGORY变量里指定的类型是否是-t选项指定的，如果是则真正运行后面的测试程序。真正运行测试程序的地方是

```
		if [ "${skip}" != "1" ]; then
			("$@" 2>&1) | tap_prefix
			local ret=${PIPESTATUS[0]}
		else
			local ret=$ksft_skip
		fi
```

所以如果你知道具体是哪个测试用例，直接运行对应的可执行文件就好。如: ./map_fixed_noreplace。

PS: 如果想增加一个测试用例可以参考文档Documentation/dev-tools/kselftest.rst，以及现有的测试用例migration.c。
