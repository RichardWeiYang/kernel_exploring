内核中的哈希表采用的是链式冲突解决方法。

# 整体的样子

```
  hlist_head
  +--------+
  |        | -> hlist_node -> hlist_node -> hlist_node
  +--------+
  |        | -> hlist_node -> hlist_node -> hlist_node
  +--------+
  |        | -> hlist_node -> hlist_node -> hlist_node
  +--------+
```

一个hash table是由多个hlist_head组成的，根据hash算法，会计算出对应的key到哪个bucket。

而每个bucket是一个以hlist_head为首，hlist_node为元素的链表。

# hlist_head链表

```

  hlist_head    hlist_node      hlist_node
  +--------+    +----------+    +----------+
  |        | +--|--pprev   | +--|--pprev   |
  |        | |  |          | |  |          |
  |  +-----|-+  |  +-------|-+  |          |
  |  |     |    |  |       |    |          |
  |  v     |    |  v       |    |          |
  |first --|--> |  next  --|--> |  next    |
  +--------+    +----------+    +----------+

```

仔细一看也算是个双向链表，不过pprev是指针的指针。


# 常用API
