有时候批量处理会给性能带来很大提升。这里总结一下相关的优化。

# 批量清理PTE

```
commit 789753e17c4d6593932f07e40b740373123296a6
Author: David Hildenbrand <david@kernel.org>
Date:   Wed Feb 14 21:44:26 2024 +0100

    mm/memory: factor out zapping of present pte into zap_present_pte()
```

# 
