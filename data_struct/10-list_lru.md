
# 长什么样

```
   struct list_lru
   +-----------------------------+
   |node                         |  这其实是一个数组，__list_lru_init中用kzalloc_objs()分配
   |    (struct list_lru_node*)  |
   |   +-------------------------+
   |   |nr_items                 |  这个比较tricky，如果是memcg上的，也会算到这里
   |   |    (atomic_long_t)      |  所以这是整个node上，链表中个数
   |   |lru                      |
   |   |    (struct list_lru_one)|
   |   |    +--------------------+
   |   |    |list                |
   |   |    |  (struct list_head)|
   |   |    |nr_items            |  list中成员的个数
   |   |    |  (long)            |
   |   |    |lock                |  保护lru的锁
   |   |    |  (spinlock_t)      |
   +---+----+--------------------+
   |list                         |  这部分只有CONFIG_MEMCG时才有
   |    (struct list_head)       |
   |shrinker_id                  |
   |    (int)                    |
   |memcg_aware                  |
   |    (bool)                   |
   |xa                           |---->struct list_lru_memcg
   |    (struct xarray)          |     +------------------------------+
   +-----------------------------+     |node[]                        |
                                       |    (struct list_lru_one)     |
                                       |    +-------------------------+
                                       |    |list                     |
                                       |    |  (struct list_head)     |
                                       |    |nr_items                 |
                                       |    |  (long)                 |
                                       |    |lock                     |
                                       |    |  (spinlock_t)           |
                                       +----+-------------------------+
```

# 使用方法

## 初始化

初始化一共有两个list_lru_init()和list_lru_init_memcg()。但最后都走到了__list_lru_init()。

其中值得注意的是node这个字段是一个数组，在__list_lru_init()中用kzalloc_objs()给每一个node分配，并通过init_one_lru()初始化。

如果配置了CONFIG_MEMCG，还可能将这个lru_list注册到memcg_list_lrus上。

## 添加/删除

添加删除分别是list_lru_add()和list_lru_del()。

## 遍历

list_lru_walk()
