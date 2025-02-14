Interval Tree(区段树)在内存管理中有用到，比如rmap和vma中。

一直觉得有点神秘，今天来看一下。

# 是附带属性的红黑树

本质上区段树是一个红黑树，在红黑树的基础上增加了两个信息：

  * leftmost: 保存了整个树中最左叶子
  * subtree:  每个节点增加当前子树内区间最大值

# 优势

和红黑树相比，区段树的优势在与查找某区间内有较差的节点时，可以借助新增的这两个信息加速查找。

具体参考区段树提供的api：

  * interval_tree_iter_first()
  * interval_tree_iter_next()

# 测试代码

内核中有相关的测试代码，lib/interval_tree_test.c。这样可以看出简单的使用方法。

