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

# 常用API
