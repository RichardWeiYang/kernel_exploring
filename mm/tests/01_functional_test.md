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

